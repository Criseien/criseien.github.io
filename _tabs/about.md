---
layout: profile
title: About
icon: fas fa-info-circle
order: 4
description: "Platform Engineer specializing in Kubernetes networking, Linux infrastructure, and reliable systems."
---

<header class="profile-hero">
  <div>
    <p class="eyebrow">About / Systems, platforms, and the path between</p>
    <h1>Platform engineering grounded in production reality.</h1>
    <p class="profile-hero__lede">
      I’m Cristian Gómez Aranda — a Platform Engineer and SRE with eight years operating critical systems at Citibanamex.
      I work where reliability, Linux infrastructure, and Kubernetes networking meet.
    </p>
  </div>
  <div class="profile-portrait">
    <img src="{{ '/assets/Imagen.jpg' | relative_url }}" alt="Cristian Gómez Aranda">
  </div>
</header>

<dl class="profile-stats" aria-label="Selected operational impact">
  <div>
    <dt>Automation</dt>
    <dd>~22,500 annual hours of manual work removed through automation</dd>
  </div>
  <div>
    <dt>Recovery</dt>
    <dd>A critical recovery path reduced from ~4 hours to 15 minutes</dd>
  </div>
  <div>
    <dt>Experience</dt>
    <dd>8 years keeping high-stakes production systems operable</dd>
  </div>
</dl>

<section class="profile-section">
  <p class="eyebrow">How I work</p>
  <h2>Start with the mechanism. Design for the person on call.</h2>
  <div class="profile-section__body">
    <p>
      My strongest work happens below the abstraction layer: the namespace, the iptables rule, the conntrack table,
      the systemd unit, and the operational decision that determines whether an incident becomes routine or catastrophic.
    </p>
    <p>
      In that environment, when something breaks at 2 AM, you may have 15 minutes before it affects millions of transactions.
      Eight years in banking taught me that infrastructure is never only technical: good platform work makes risk visible,
      removes unnecessary manual effort, and gives engineers a safe path to move faster.
    </p>
    <p>
      That depth is a deliberate choice, not a gap: I go past the managed-cloud abstraction layer — the Helm chart, the
      console click, the Terraform apply — on purpose, because understanding what is actually running underneath a
      platform is what turns an incident into something routine instead of something mysterious. Most platforms are
      operated by people who stopped at the abstraction. I kept going.
    </p>
    <p>
      That depth doesn't stay solo, either. Inside the bank, I built and delivered internal training on production
      operations and AI-assisted automation tooling — video walkthroughs, automation guides, Copilot usage guides — and
      ran live sessions for the wider production team. None of that material is public; it lived inside the bank's
      internal systems. But the habit is the same one behind this site: understand something deeply, then make it
      legible for the next person.
    </p>
  </div>

  <div class="focus-grid">
    <div>
      <h3>Kubernetes networking</h3>
      <p>Namespaces, veth pairs, iptables, kube-proxy, CoreDNS, CNI fundamentals, and the full packet path.</p>
    </div>
    <div>
      <h3>Linux systems</h3>
      <p>RHEL-family operating systems, cgroups, systemd, containers, and the primitives that make a platform run.</p>
    </div>
    <div>
      <h3>Reliable operations</h3>
      <p>Automation, observability, incident response, and infrastructure decisions that are traceable and defensible.</p>
    </div>
  </div>
</section>

<section class="profile-section">
  <p class="eyebrow">Public work</p>
  <h2>Evidence over a tool list.</h2>
  <div class="proof-list">
    <a href="https://limen.icris.me/">
      <strong>Banco Limen — a fictional on-premise bank shaped by recurring production patterns</strong><span>↗</span>
    </a>
    <a href="{{ '/' | relative_url }}#writing">
      <strong>From Scratch — Kubernetes networking explained from Linux primitives</strong><span>↗</span>
    </a>
    <a href="https://github.com/Criseien/platform-fundamentals" target="_blank" rel="noopener noreferrer">
      <strong>Platform fundamentals — archived, hands-on reference work from kernel to orchestration</strong><span>↗</span>
    </a>
  </div>
</section>

<section class="profile-section profile-now">
  <p class="profile-now__label">Now / 2026–27</p>
  <div>
    <h2>Preparing for AI workload infrastructure.</h2>
    <p>
      I’m deepening my Kubernetes networking and platform fundamentals through hands-on projects and public writing.
      Next, I’m studying how those foundations support secure, observable AI workloads through model serving,
      multi-tenancy, network isolation, and operational evidence.
    </p>
  </div>
</section>

<section class="profile-contact" aria-label="Connect">
  <p>Open to remote Platform Engineer and SRE roles with teams running Kubernetes in production — based in Mexico City, strong overlap with US business hours.</p>
  <a class="button button--primary" href="mailto:agcristianaranda@icloud.com">Email me <span aria-hidden="true">↗</span></a>
  <a class="button button--quiet" href="https://www.linkedin.com/in/cristiangomezaranda/" target="_blank" rel="noopener noreferrer">LinkedIn <span aria-hidden="true">↗</span></a>
  <a class="button button--quiet" href="{{ '/assets/cv-cristian-gomez-aranda.pdf' | relative_url }}" download>Download CV <span aria-hidden="true">↓</span></a>
</section>
