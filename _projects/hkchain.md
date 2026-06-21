---
layout: page
title: Compliance-Native L1
description: A Hong Kong–regulated, compliance-native Layer 1 for stablecoin issuance and real-world-asset tokenization.
img: assets/img/projects/hkchain-cover.jpg
importance: 2
category: work
related_publications: false
---

HKChain is a Hong Kong–regulated, compliance-native Layer 1, built on the Cosmos SDK with
EVM compatibility and aimed at regulated stablecoin issuance and real-world-asset (RWA)
tokenization. I contribute to its technical architecture.

The core idea is to enforce compliance at the **infrastructure layer** rather than in
application code. Identity, KYC, and policy constraints are checked as protocol-level
invariants *before* a transaction is admitted — not discovered after the fact. Compliance
becomes a property of the chain itself, instead of something every application has to
re-implement and be trusted to get right.

#### Compliance enforced by the chain

The clip below walks through that admission path: a transaction arrives, the protocol checks
it against identity and policy rules, and only a compliant transaction settles — a
non-compliant one is rejected at the gate.

<div style="position:relative;width:100%;padding-top:56.25%;border-radius:10px;overflow:hidden;box-shadow:0 8px 30px rgba(0,0,0,.15);margin:1.5rem 0;">
  <iframe src="{{ '/assets/html/hkchain-compliance-30s.html' | relative_url }}"
          loading="lazy" title="HKChain — compliance admission flow"
          style="position:absolute;inset:0;width:100%;height:100%;border:0;"></iframe>
</div>

#### On top: real-world assets

With that base in place, regulated assets can live on-chain. This second clip follows one
applied case — tokenized green-energy devices settling their output on the same
compliance-native rails.

<div style="position:relative;width:100%;padding-top:56.25%;border-radius:10px;overflow:hidden;box-shadow:0 8px 30px rgba(0,0,0,.15);margin:1.5rem 0;">
  <iframe src="{{ '/assets/html/hkchain-gogreen-30s.html' | relative_url }}"
          loading="lazy" title="HKChain × GoGreen — RWA settlement"
          style="position:absolute;inset:0;width:100%;height:100%;border:0;"></iframe>
</div>

**Stack:** Cosmos SDK · EVM-compatible execution · protocol-layer compliance modules.
