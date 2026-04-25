---
title: "Exposing a service without Kubernetes: DNAT load balancing with iptables"
date: 2026-04-25 14:00:00 -0600
categories: [Linux, Kubernetes]
tags: [networking, namespaces, nat, dnat, iptables, conntrack, load-balancing, kube-proxy, k8s]
description: "How to expose a container to the outside world and distribute traffic across multiple backends using iptables DNAT — the same mechanism kube-proxy uses for NodePort and ClusterIP services."
---

## The problem

You built the network: namespaces connected through `br0`, outbound traffic working via MASQUERADE. But traffic only flows *out*. An external client hitting your host's IP gets nothing back — the kernel has no idea what to do with it.

This article solves the inbound side. You'll expose a service running inside a namespace to the outside world, then extend it to load balance across three backends. No Kubernetes, no proxy. Just iptables.

---

## What you already have

From the previous articles:

- `ns-blue` (`10.0.0.1`) and `ns-green` (`10.0.0.2`) connected through `br0`
- `ip_forward` enabled
- MASQUERADE handling outbound NAT

You'll add a third namespace for the load balancing example:

```bash
ip netns add ns-red

ip link add veth-red type veth peer name veth-red-br
ip link set veth-red netns ns-red
ip link set veth-red-br master br0

ip netns exec ns-red ip addr add 10.0.0.3/24 dev veth-red
ip netns exec ns-red ip link set veth-red up
ip link set veth-red-br up
```

Start a web server inside each namespace. `ncat` works fine for testing:

```bash
ip netns exec ns-blue  bash -c 'while true; do echo -e "HTTP/1.0 200 OK\r\n\r\nblue"  | ncat -l 80; done' &
ip netns exec ns-green bash -c 'while true; do echo -e "HTTP/1.0 200 OK\r\n\r\ngreen" | ncat -l 80; done' &
ip netns exec ns-red   bash -c 'while true; do echo -e "HTTP/1.0 200 OK\r\n\r\nred"   | ncat -l 80; done' &
```

---

## Step 1: DNAT for a single backend

MASQUERADE rewrites the *source* of outbound packets. **DNAT** (Destination NAT) does the reverse — it rewrites the *destination* of inbound packets before the kernel decides where to route them.

The rule sits in the `nat` table, `PREROUTING` chain — evaluated the moment a packet arrives on an interface, before routing:

```bash
# redirect external traffic on port 8080 to ns-blue's web server
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8080 \
  -j DNAT --to-destination 10.0.0.1:80
```

With just this rule, traffic is redirected but silently dropped. The `FORWARD` chain is responsible for packets being routed between interfaces, and its default policy is typically `DROP`. You need to explicitly allow it:

```bash
# allow forwarded traffic to reach ns-blue
iptables -A FORWARD -d 10.0.0.1 -p tcp --dport 80 -j ACCEPT
```

Test from an external machine:

```bash
curl http://<host-ip>:8080
# blue
```

---

## What conntrack does here

Same mechanism as MASQUERADE, but in reverse. When the DNAT rule fires, conntrack records the translation:

```
client_ip:port  →  host_ip:8080  rewrites to  10.0.0.1:80
```

When `ns-blue` responds, the kernel looks up the conntrack table, finds the original destination (`host_ip:8080`), rewrites the source of the *response* back to that address, and delivers it to the client. The client never sees `10.0.0.1` — it only ever talks to the host's IP.

```bash
# watch the translation in real time
conntrack -L | grep 8080
```

---

## Step 2: Load balancing across three backends

A single DNAT rule sends all traffic to one backend. To distribute it, you chain multiple DNAT rules with the `statistic` module. Each rule fires with a calculated probability so that over many connections, the load is split evenly.

The math works like this: the first rule catches 1 in 3 connections. Of the remaining 2/3, the second rule catches half (1/3 overall). The third rule catches everything left (1/3 overall).

```bash
# remove the single-backend DNAT rule from Step 1
iptables -t nat -D PREROUTING -i eth0 -p tcp --dport 8080 \
  -j DNAT --to-destination 10.0.0.1:80
 

# 33% → ns-blue
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8080 \
  -m statistic --mode random --probability 0.33 \
  -j DNAT --to-destination 10.0.0.1:80

# 50% of remaining (~33% overall) → ns-green
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8080 \
  -m statistic --mode random --probability 0.50 \
  -j DNAT --to-destination 10.0.0.2:80

# all remaining (~33% overall) → ns-red
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8080 \
  -m statistic --mode random --probability 1 \
  -j DNAT --to-destination 10.0.0.3:80

# allow FORWARD to other backends 
iptables -A FORWARD -d 10.0.0.2 -p tcp --dport 80 -j ACCEPT
iptables -A FORWARD -d 10.0.0.3 -p tcp --dport 80 -j ACCEPT
```

Verify the distribution:

```bash
for i in $(seq 1 9); do curl -s http://<host-ip>:8080; echo; done
# blue
# green
# red
# blue
# red
# green
# ...
```

Verify the rules are in place:

```bash
iptables -t nat -L PREROUTING -n -v --line-numbers
```

---

## The full picture

![DNAT load balancing: external traffic arrives on eth0 port 8080, iptables PREROUTING applies DNAT to redirect to one of three namespaces (ns-blue, ns-green, ns-red) through br0, conntrack tracks each translation for the return path.](/assets/img/dnat-load-balancing.png)

The path an inbound packet takes:

1. External client sends a request to `host-ip:8080`
2. Packet arrives on `eth0`, hits the `PREROUTING` chain
3. The `statistic` module fires — one of the three DNAT rules matches
4. Kernel rewrites destination to `10.0.0.x:80`
5. Conntrack records: `client_ip:port → host_ip:8080 ↔ 10.0.0.x:80`
6. Packet is forwarded through `br0` into the target namespace
7. The web server responds — conntrack rewrites the source back to `host_ip:8080`
8. Client receives the response as if it came from the host

---

## This is exactly what kube-proxy does

When you create a `Service` in Kubernetes, kube-proxy (in iptables mode) writes DNAT rules to the node — one per endpoint, chained with the `statistic` module, the same way you just did it manually.

For a `ClusterIP` service, traffic sent to the virtual IP gets DNAT'd to a pod IP. For `NodePort`, traffic arriving on the node's IP at the assigned port gets the same treatment, just with an extra rule in `PREROUTING` that catches the external port first.

The difference between what you built here and what kube-proxy manages: kube-proxy updates those rules automatically every time a pod starts or dies. The mechanism is identical.

Next time a `Service` isn't routing traffic correctly, you have a checklist:

- Do the DNAT rules exist? In a real Kubernetes node, kube-proxy doesn't write rules directly to `PREROUTING` — it jumps to custom chains (`KUBE-SERVICES` → `KUBE-SVC-XXX` → `KUBE-SEP-XXX`). Filter by ClusterIP instead: `iptables-save -t nat | grep <ClusterIP>`
- Is the FORWARD chain allowing the translated destination? `iptables -L FORWARD -n -v`
- Is conntrack tracking the connection? `conntrack -L | grep <port>`
- Is the pod actually listening on the target port? `ip netns exec <ns> ss -tlnp`

Start there.
