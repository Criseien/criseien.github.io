---
title: "How pods reach the internet: NAT and masquerading"
date: 2026-03-27 14:00:00 -0600
categories: [Linux, Kubernetes]
tags: [networking, namespaces, nat, masquerade, iptables, conntrack, containers, k8s]
description: "Why pods can't reach the internet by default and how iptables MASQUERADE fixes it — a deep dive into NAT, conntrack, and the path packets take out of a Kubernetes node."
---

## The problem

Your namespace can reach other namespaces on the same host — you built that in the [previous article](/posts/building-container-network-from-scratch/). But try running `curl https://api.github.com` from inside one and you get nothing. The namespace is isolated. It has no path out.

This article explains exactly how that exit path works, from the Linux primitives up.

---

## What you already have

From the previous article:

- `ns-blue` and `ns-green` connected through `br0`
- Traffic between namespaces works
- Neither namespace can reach the internet

The bridge handles traffic *between* namespaces on the same host. To get traffic *out* — toward the internet — you need two more things: **IP forwarding** and **NAT**.

---

## Step 1: IP forwarding

By default, Linux does not forward packets between network interfaces. It drops them. This is a security default — a machine that isn't a router shouldn't behave like one.

The setting that controls this:

```bash
# check current state — 0 means disabled
sysctl net.ipv4.ip_forward
```

To enable it:

```bash
# temporary — resets on reboot
sysctl -w net.ipv4.ip_forward=1

# permanent
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
sysctl -p

# verify
sysctl net.ipv4.ip_forward
# net.ipv4.ip_forward = 1
```

With forwarding enabled, the kernel will pass packets from `br0` to `eth0`. But there's still a problem: the packet leaving through `eth0` carries the namespace's private IP as its source address — `10.0.0.1`. The internet has no idea how to route a response back to a private address. That's where NAT comes in.

---

## Step 2: NAT and MASQUERADE

NAT (Network Address Translation) rewrites the source IP of outbound packets so they appear to come from your host's public IP. When the response arrives, NAT translates the destination IP back to the original private address and delivers it to the right namespace.

There are two ways to configure source NAT in iptables:

- **MASQUERADE** — uses whatever IP `eth0` currently has. Ideal when your external IP is assigned by DHCP and can change.
- **SNAT** — you specify a fixed source IP explicitly. Used when your external IP is static.

For most scenarios — and for how Kubernetes handles it — MASQUERADE is the right choice:

```bash
# apply MASQUERADE to all traffic leaving through eth0
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# verify the rule is active
iptables -t nat -L POSTROUTING -n -v
```

---

## What happens behind the scenes: conntrack

When a packet leaves through NAT, the kernel doesn't just rewrite the source IP and forget it. It records the translation in the **conntrack** table — a kernel state table that maps:

```
private_ip:port  ↔  public_ip:port
```

When the response arrives from the internet, the kernel looks up that table, finds the original sender, rewrites the destination back to the private address, and delivers it to the right namespace.

```bash
# list active conntrack entries
conntrack -L

# filter for established connections only
conntrack -L | grep ESTABLISHED
```

Without conntrack, NAT would be one-way. Responses would arrive at the host with no record of where they belong.

---

## The full picture

![Full NAT and masquerade flow: namespace → veth → bridge → ip_forward → eth0 → Internet, with MASQUERADE and conntrack.](/assets/img/nat-masquerade-flow.png)

The path an outbound packet takes:

1. Process inside `ns-blue` sends a packet to `8.8.8.8`
2. Packet travels through `veth-blue` to `br0`
3. Kernel checks: is `ip_forward` enabled? Yes → forwards to `eth0`
4. iptables POSTROUTING: MASQUERADE rewrites source IP to `eth0`'s IP
5. Conntrack records the mapping: `10.0.0.1:5432 ↔ 203.0.113.5:443`
6. Packet leaves the host toward the internet
7. Response arrives at `eth0` with destination = host's public IP
8. Conntrack matches it → rewrites destination back to `10.0.0.1`
9. Packet is delivered back to `ns-blue`

---

## This is exactly what Kubernetes does

When a pod sends traffic outside the cluster, the CNI plugin has already set up veth pairs and a bridge. `ip_forward` is enabled on the node. And there's a MASQUERADE rule in the `nat` table rewriting the pod's IP to the node's IP before it hits the network — or the eBPF equivalent in Cilium.

Next time a pod can't reach an external service, you have a checklist:

- Is `net.ipv4.ip_forward` enabled on the node?
- Does the MASQUERADE rule exist in the `nat` table?
- Is conntrack tracking the connection?

Start there.
