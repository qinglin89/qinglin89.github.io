---
layout: page
title: QuantX
description: A Go-based quantitative trading platform — systematic strategies with backtest-to-live parity and a human-AI research loop.
img: assets/img/projects/quantx-cover.jpg
importance: 1
category: work
related_publications: false
---

QuantX is a quantitative trading platform I design and build solo, in Go. It runs systematic
strategies, and its guiding goal is correctness above all — results that are reproducible,
verifiable, and grounded in real execution constraints rather than idealized assumptions.

#### One code path, four environments

The same strategy code runs unchanged across **backtest, paper, demo, and live**. The stack is
built so that what a strategy does in a backtest is what it does in production: execution
semantics — order types, fills, funding, liquidation — are modeled closely enough to real
exchange behavior that backtest-to-live parity *holds* rather than quietly drifts.

#### Human-defined priors, AI-assisted research

Strategy research runs as a loop between two roles. I define the structural priors — the
heuristics, constraints, and economic intuitions that shape where it is worth looking. An AI
research layer then explores within those boundaries, proposing and refining candidates. The
human stays in the loop by design: the AI is a research accelerator operating inside a defined
frame, not a black box left to surface signals on its own.

#### Validation over conviction

A candidate means nothing until it survives validation. Ideas are tested against outcomes —
predictive-power checks, out-of-sample behavior, and the same backtest-to-live parity that
governs execution — rather than against opinion. The product framing follows from this: QuantX
is about disciplined, risk-managed systematic exposure, not promises of alpha.

**Stack:** Go · four-tier execution (backtest · paper · demo · live) · live trading infrastructure.
