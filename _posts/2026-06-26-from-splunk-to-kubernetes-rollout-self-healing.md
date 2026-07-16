---
title: "From Splunk to Kubernetes: what 8 years of alert fatigue taught me about self-healing systems"
date: 2026-06-26 12:00:00 -0600
categories: [Kubernetes, SRE]
tags: [kubernetes, sre, rollout, deployment, self-healing, k8s, devops, imagepullbackoff, rolling-update]
description: "Eight years running production SRE at a major bank taught me one thing about alert fatigue: detection is the hard part. Kubernetes changed that assumption for me today."
---

For eight years I worked as an SRE at a major financial institution. Legacy services, high-stakes transactions, multiple platforms running simultaneously. Monitoring coverage was extensive — Splunk, AppDynamics, and several others feeding dashboards around the clock. Thousands of alerts per day.

## The problem with detecting your own failures

The architecture to manage that alert volume was layered. Each domain had its own distribution list with pre-filtered, high-priority alerts routed directly to the responsible team. The intent was good: reduce noise, surface what matters, close the gap between failure and response.

At the scale of a large institution, across multiple shifts and dozens of services running simultaneously, alerts still got buried. An incident would start silently. By the time it was detected, triaged, and routed to the right specialist team, a service had been failing for minutes. Sometimes longer.

Then the real work started. In production at a bank, you don't just restart a service. You open an incident, notify stakeholders, get a specialist on a call. If the issue came from a bad deployment — say, someone pushed a wrong image — you needed an **urgent change request with approvals** to roll it back. Paperwork, signatures, a second pair of eyes. All while clients were impacted.

Once approved, the team would open the internal Ansible automation platform, select the right playbook — restart httpd, restart the Java service, restart the JVM — and run it. Watch the console output confirm the service came back. Then switch to the monitoring dashboards to verify recovery. Then close the incident.

The system worked. It just required humans at every step: to notice, to escalate, to approve, to run the automation, to verify. Even with every team already on an incident bridge, that cycle is measured in hours, not minutes. And every minute between failure and detection is a minute of client impact — on a payment outage, that impact is measurable in transaction volume, not just inconvenience.

## Today, my rollout broke — and I didn't have to notice

I'm currently deep in hands-on Kubernetes and platform engineering practice. During a drill today, I pushed an image update using a tag I hadn't verified:

```bash
kubectl set image deployment api httpd=httpd:2.4.1 --namespace=drill-fri
kubectl rollout status deployment api --namespace=drill-fri
```

The rollout stalled. `httpd:2.4.1` doesn't exist on Docker Hub. Three of five pods went into `ImagePullBackOff`. After ten minutes:

```
error: deployment "api" exceeded its progress deadline
```

Here's what I noticed when I checked the pods:

```bash
kubectl get pods --namespace=drill-fri
```

```
NAME                   READY   STATUS             RESTARTS   AGE
api-57ffdf6c9b-k77xl   0/1     ImagePullBackOff   0          20m
api-57ffdf6c9b-lq98m   0/1     ImagePullBackOff   0          20m
api-57ffdf6c9b-xxksp   0/1     ImagePullBackOff   0          20m
api-5f4768f975-9dtr2   1/1     Running            0          45m
api-5f4768f975-qgt98   1/1     Running            0          47m
api-5f4768f975-slxz9   1/1     Running            0          47m
api-5f4768f975-wcmfw   1/1     Running            0          47m
```

Four old pods were still running. The bad update never replaced them.

## What's actually happening under the hood

Kubernetes `RollingUpdate` — the default deployment strategy — has two key parameters:

- **`maxUnavailable`**: how many pods can be down simultaneously during an update. Default: 25%. Kubernetes rounds **down** — with 5 replicas, that's 1 pod unavailable.
- **`maxSurge`**: how many extra pods above the desired count can exist during an update. Default: 25%. Kubernetes rounds **up** — with 5 replicas, that's 2 extra pods, giving a ceiling of 7 total.

When the rollout started, Kubernetes spun up one new pod (`httpd:2.4.1`), waited for it to become `Ready`, and — when it didn't — refused to terminate the old ones. It tried a second pod. A third. Each one failed to pull the image. After `progressDeadlineSeconds` (600 seconds by default), the rollout gave up and reported an error.

The old pods never went down. There was no human alert. No incident. No approval chain.

## Diagnosing it

Once I saw the error, the diagnostic sequence was straightforward:

```bash
# 1. Check pod states
kubectl get pods --namespace=drill-fri

# 2. Describe a failing pod to see Events
kubectl describe pod api-57ffdf6c9b-k77xl --namespace=drill-fri
```

The Events section said everything:

```
Failed to pull image "httpd:2.4.1": ... docker.io/library/httpd:2.4.1: not found
```

Invalid image tag. Recovery was one command:

```bash
kubectl rollout undo deployment api --namespace=drill-fri
kubectl rollout status deployment api --namespace=drill-fri
# deployment "api" successfully rolled out
```

No approval. No incident bridge. No phone call.

## What eight years taught me

The gap between Ansible automation and Kubernetes isn't the automation itself — it's **where detection happens**. In the old model, a human had to notice the failure before the automation could run. The faster that human noticed, the shorter the blast radius.

Kubernetes moves detection upstream. The platform checks whether the new version is healthy *before* it decommissions the old one. `maxUnavailable` is not just a configuration parameter — it's a policy that says: *do not proceed until the replacement is proven working*. The rollout stalling wasn't a failure. It was the system making the right decision without being asked.

I spent eight years building the human layer on top of reactive monitoring. What I'm learning now is a different model: encode the health check into the deployment itself, and let the platform refuse the bad update before anyone has to notice.

## When you see ImagePullBackOff on a rollout

```
1. kubectl get pods --namespace=<ns>
   → Look for ImagePullBackOff on the NEW pods, Running on the OLD ones.
   → If old pods are Running: maxUnavailable protected you. No client impact.

2. kubectl describe pod <failing-pod> --namespace=<ns>
   → Read the Events section. The exact error is there.
   → Common causes: tag doesn't exist, registry unreachable, pull secret missing.

3. kubectl rollout undo deployment <name> --namespace=<ns>
   → Reverts to the previous ReplicaSet. Verify with rollout status.

4. Fix the image tag, update your CI pipeline, re-deploy.
```

The cluster will tell you what's wrong. You just need to know where to look.

---

**Banco Limen:** A fictional on-premise bank shaped by recurring production patterns. [Explore the Phase 1 baseline →](https://limen.icris.me/)
