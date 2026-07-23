+++
title = "HMC and NUTS: Smart Trajectory Construction"
description = "Work-in-progress slides on the No-U-Turn Sampler as dynamic Hamiltonian trajectory construction."
date = "2026-07-23"
summary = "Work-in-progress slides on HMC, NUTS, U-turn checks, tree building, and dynamic trajectory construction."
tags = ["HMC", "NUTS", "MCMC", "presentations"]
+++

<style>
  .presentation-status {
    align-items: center;
    background: var(--code-bg);
    border: 1px solid var(--border);
    border-radius: 6px;
    display: inline-flex;
    font-size: 0.85rem;
    font-weight: 650;
    gap: 0.4rem;
    margin: 0 0 1rem;
    padding: 0.35rem 0.65rem;
  }

  .presentation-frame {
    aspect-ratio: 16 / 9;
    border: 1px solid var(--border);
    border-radius: 6px;
    margin: 1.25rem 0 0.75rem;
    overflow: hidden;
    width: 100%;
  }

  .presentation-frame iframe {
    border: 0;
    display: block;
    height: 100%;
    width: 100%;
  }

  .presentation-actions {
    display: flex;
    flex-wrap: wrap;
    gap: 0.75rem;
    margin-bottom: 1.25rem;
  }
</style>

<div class="presentation-status">Work in progress</div>

These slides are an early version of a talk on HMC and NUTS, focusing on trajectory construction, U-turn checks, dyadic tree expansion, and proposal selection.

<div class="presentation-actions">
  <a href="/slides/nuts-smart-trajectory/">Open fullscreen slides</a>
</div>

<div class="presentation-frame">
  <iframe src="/slides/nuts-smart-trajectory/" title="HMC and NUTS: Smart Trajectory Construction slides" loading="lazy" allowfullscreen></iframe>
</div>

## Sources and Further Reading

- Hoffman and Gelman, [The No-U-Turn Sampler: Adaptively Setting Path Lengths in Hamiltonian Monte Carlo](https://jmlr.org/papers/v15/hoffman14a.html).
- Betancourt, [A Conceptual Introduction to Hamiltonian Monte Carlo](https://arxiv.org/abs/1701.02434).
- Durmus, Gruffaz, Kailas, Saksman, and Vihola, [On the convergence of dynamic implementations of Hamiltonian Monte Carlo and No U-Turn Samplers](https://arxiv.org/abs/2307.03460).
