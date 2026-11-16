# Working rules for this repository

The full specification is `docs/BRIEF.md`. Progress is tracked in `docs/PLAN.md`. Read both at the start of every session and continue from the first unchecked phase.

## Who you are working with

The owner is a product/project manager with limited coding experience and no local development machine. He works only through Claude Code on the web and the GitHub website.

## Rules

1. One phase at a time, one pull request per phase. Stop after each phase and wait.
2. Every PR description has: what was built, why, and a "How to check" list that works from a browser.
3. Update the checkboxes in `docs/PLAN.md` in the same PR.
4. Explain important files and decisions in plain language. No unexplained config values.
5. Never invent results. Numbers in docs and reports must come from a reproducible measurement.
6. Do not guess library versions, API details or prices. Check the documentation or ask.
7. No secrets in the repo. Tests and CI use the fake LLM client and need no API key.
8. Do not assume Docker or a cluster exists in your sandbox. Cluster checks run in GitHub Actions with kind.
9. Pipeline stages and prompts live in `config/pipelines/`. Design tokens live in one file. Prices live in `config/pricing.yaml`. Keep them out of code.
10. Core scope before Stretch scope (BRIEF section 10).
11. If the owner must act outside the repo, give exact copy-paste or click-by-click steps.
12. If a request is a bad idea, say so and propose an alternative.
13. Record significant decisions in `docs/adr/`.
