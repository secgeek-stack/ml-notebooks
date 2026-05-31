# Reward Hacking and Hardening Notebook

A small-scale empirical study of reward model vulnerability and hardening in a mobile security threat assessment domain.

## What this is

Reward models are trained judges. Given two competing assessments of the same evidence, they learn which one is better — and that learned preference can become a training signal for a downstream model.

The problem: a reward model trained on too few examples, or on examples with surface-level correlations, can be fooled. A wrong assessment dressed in confident technical language can score higher than a correct one without the reasoning improving. That is reward hacking.

This project demonstrates that mechanism at small scale, using mobile threat assessment as the domain, and shows that targeted adversarial data augmentation reduces (but does not eliminate) the vulnerability.

## What the notebook does

1. Trains two reward models on `pairs_baseline.json` — 5 preference pairs, no adversarial examples
2. Demonstrates that both models are exploitable by surface features: technical jargon, length padding, and confident framing
3. Retrains both models on `pairs_hardened.json` — 8 pairs including 3 adversarial examples
4. Measures whether hardening reduced exploitability

Two model architectures are compared:
- **Pointwise** (logistic regression on sentence embeddings): scores each assessment independently
- **Pairwise** (Bradley-Terry loss): always sees chosen and rejected side by side for the same telemetry, directly optimizing the margin between them

## Data

8 synthetic preference pairs grounded in realistic Android telemetry signals: PackageManager install events, AppOps operation counts, permission grants, AccessibilityService events, network egress with DNS traces, execution context (foreground/background, user interaction, screen state, battery spike), IPC calls, and signing certificate metadata.

Each pair has:
- `telemetry`: structured JSON event chain
- `chosen`: the better assessment — grounded in the telemetry signals
- `rejected`: the worse assessment — false negative, false positive, right verdict wrong reason, miscalibrated severity, or subtly worse
- `preference_strength`: clear or subtle — used to weight the pairwise loss

**Scope note:** This is an approximation of what a lightweight on-device monitor would collect without root access (Android AppOps, PackageManager, JobScheduler, NetworkStats APIs). Real EDR agents on mobile collect additional signals including process trees, syscall traces, Binder IPC, and DEX/bytecode hashes that require kernel-level access or vendor partnerships. All packages use `.test` domains and `com.example.*` namespaces. No real malware samples were used.

## Key finding

Training on 5 pairs produced a pairwise reward model where technical jargon alone moved the score from -2.107 (clearly rejected territory) to +0.720 (chosen territory) — a delta of +2.827 — without the underlying reasoning improving. The assessment still missed an SMS exfiltration attack. It just sounded authoritative.

Adding 3 adversarial pairs where verbose jargon-heavy wrong assessments were explicitly penalized kept the same hack in rejected territory (-0.301) after hardening. The hack still moved the score substantially (+3.625 delta) but no longer crossed the decision boundary.

The fix is in the data, not the loss function. Neither model is robust without adversarial pairs that break the style-quality correlation.

## What this does not demonstrate

The downstream training consequence: a policy model learning to produce jargon-heavy wrong assessments because the reward model rewards them. That requires a generative policy model and a full RL training loop. Named here as the natural next step.

## Relation to prior work

This notebook is a domain-specific, small-scale instantiation of the adversarial training paradigm described in Bukharin et al. (2025). Where they use a learned policy to automatically generate adversarial examples against general-purpose reward models at scale, this work uses security domain expertise to hand-author adversarial preference pairs targeting semantically meaningful failure modes in mobile threat assessment — specifically the conflation of technical register with reasoning quality. The hardening mechanism is the same: expose the vulnerability, then close it through targeted data augmentation.

## Files

```
secreward.ipynb          — main notebook
pairs_baseline.json      — 5 original preference pairs
pairs_hardened.json      — 8 pairs including 3 adversarial examples
```

## Setup

```bash
pip install sentence-transformers torch scikit-learn
```

Run the notebook cells in order. Upload both JSON files to your Colab session before running Cell 2.

## References

Bukharin et al. (2025). *Adversarial Training of Reward Models.*
arXiv:2504.06141. https://arxiv.org/abs/2504.06141

Gao et al. (2022). *Scaling Laws for Reward Model Overoptimization.*
arXiv:2210.10760. https://arxiv.org/abs/2210.10760

Sharma et al. (2024). *Towards Understanding Sycophancy in Language Models.*
ICLR 2024. https://arxiv.org/abs/2310.13548
