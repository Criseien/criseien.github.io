---
# the default layout is 'page'
title: Banco Limen
icon: fas fa-landmark
order: 5
published: true
description: "A Phase 1 reconstruction of legacy banking infrastructure failure modes and the decisions behind their eventual repair."
---

*Limen* is Latin for threshold: the space between what was and what could be.

Banco Limen is a fictional, composite retail bank. It is a controlled Phase 1 reconstruction of generalized failure patterns from regulated production — not an employer system, source codebase, or data set. The app is the pretext; the infrastructure around it, and the reasoning behind its eventual repair is the point.

> **Current status:** Phase 1 is in progress. The stack deliberately exposes legacy anti-patterns so they can be inspected before later phases address them. This is not a finished cloud-native case study.

## The current Phase 1 architecture

All components share one flat Docker bridge network. That lack of segmentation is intentional: it is part of the system being examined.

```text
                                Banco Limen / Phase 1

  Client
    │ :5001
    ▼
  Flask application (:5000) ───────────────► PostgreSQL 12
    │  exposes /metrics                         │
    │                                            │
    ├──── Prometheus scrapes ────────────────────┘
    │          │
    │          ├────► Grafana reads metrics
    │          ├────► node-exporter (running)
    │          └────► postgres-exporter + legacy VMs (configured, absent)
    │
    ├────► Elasticsearch ◄──── Kibana
    │       (no retention)      (stale dashboards)
    │
    └────► Jaeger
            installed, but the application emits no spans

  All services: one flat limen-network bridge
```

The architecture is intentionally inconsistent: Prometheus, ELK, and Jaeger coexist without useful correlation; the database has no replica; several scrape targets do not exist; and a compromised service can reach every component on the flat network.

## What to inspect

1. [Problem inventory](https://github.com/Criseien/Limen/blob/main/docs/problems/legacy-problems.md) — 38 documented failure patterns.
2. [Phase 1 design](https://github.com/Criseien/Limen/blob/main/docs/architecture/phase1-design.md) — the current stack and the anti-patterns it reproduces.
3. [Architecture Decision Records](https://github.com/Criseien/Limen/tree/main/docs/decisions) — why each deliberate decision exists in Phase 1.
4. [The Limen repository](https://github.com/Criseien/Limen) — source, current roadmap, and the preserved Phase 1 baseline.

## Related reading

- [Building a container network from scratch: namespaces, veth, bridges](/posts/building-container-network-from-scratch/)
- [How pods reach the internet: NAT and masquerading](/posts/how-pods-reach-internet-nat-masquerading/)
- [Exposing a service without Kubernetes: DNAT load balancing with iptables](/posts/exposing-service-dnat-load-balancing-iptables/)
- [From Splunk to Kubernetes: what alert fatigue taught me about safer rollouts](/posts/from-splunk-to-kubernetes-rollout-self-healing/)
