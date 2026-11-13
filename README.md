# AI-Native SDLC Pipeline

A guide to running software development with AI agents without losing control of architecture and quality.

Code generation is no longer the bottleneck. Human attention and early-stage errors are. This guide describes a pipeline that moves engineering effort to where it matters: capturing context, approving designs, enforcing quality gates deterministically, and spending human review only where the risk justifies it.

## Documents

- **[Overview](docs/overview.md)**: the approach itself.
  - Tiered multi-phase pipeline (Inreach → Research → Design → QA Plan → Plan → Code → Q-Check → Deploy & Test → Release), with explicit backtracking rules
  - Quality gates: deterministic checks, calibrated AI judges, and targeted human approval
  - Mutation testing, CRAP score, SAST/DAST
  - Review strategy routed by DDD domain × risk tags, with sampled audits
  - Feedback loop on DORA and pipeline metrics, agent security, and an adoption checklist
- **[Tech Stack](docs/tech-stack.md)**: tooling to enforce and monitor each mechanism in the overview, plus a pilot and scale-up rollout plan.

## Key ideas

- **Avoid the "Meat Proxy".** Engineers must stay the ones who understand the system, not just pass work between agents.
- **Catch errors early.** A mistake in context or design cascades through everything downstream, so humans approve those phases.
- **Specs constrain code, never the reverse.** QA contracts are frozen; a needed change goes back to Design.
- **Gates must be ungameable.** Agents can't edit thresholds, suppress checks, or skip tests.
- **Spend human attention by risk.** Core domain and risk-tagged changes get deep review; generic code is auto-approved and audited by sampling.

## Status

Draft. Thresholds and tier boundaries are starting points to calibrate per team (see *Open Decisions* in the overview), and the tech stack has open questions about platform and infrastructure.
