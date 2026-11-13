# Tech Stack for the AI-Native SDLC Pipeline

This document maps each mechanism described in [overview.md](overview.md) to tooling that can enforce it and monitor it.

**Assumptions:** GitHub as the Git platform, with GitLab equivalents noted where they differ. The language, infrastructure, and hosting model are not decided yet (see [Open Questions](#open-questions)). Tool capabilities and licensing change quickly, so verify current versions before adopting a tool.

## Architecture principle

The pipeline is split into three planes, and agents are never in charge of the flow:

| Plane | Responsibility | Primary tools |
|---|---|---|
| **Orchestration** | Holds each task's state: tier, current phase, retries, backtracks, and pending human approvals. Calls agents as workers. | Temporal (target), GitHub Actions + labels (pilot) |
| **Enforcement** | Makes the rules impossible to skip: required checks, required reviewers, protected files. | GitHub Rulesets, CODEOWNERS, OPA/Conftest, Danger |
| **Observability** | Records every phase transition, gate result, agent action, and deployment outcome, and feeds the feedback loop. | OpenTelemetry, Langfuse, Apache DevLake, Grafana |

All gate thresholds live in one versioned file (for example `gates.yaml`) protected by CODEOWNERS. Changing a threshold is a pull request, so "tightening is automatic, loosening is a human decision" (overview §5) is enforced by the same review mechanism as everything else.

## 1. Orchestration and agent execution

Covers overview §2 (phases, tiers, backtracking, artifact lifecycle).

| Need | Recommended | Alternatives |
|---|---|---|
| Durable phase state, retries, backtracking, waiting on human approvals | **Temporal**: one workflow per task, one activity per phase, human approvals as signals | LangGraph (checkpoints, interrupts). Pilot only: GitHub Actions + PR labels as the state machine |
| Agent runtime per phase | **Claude Agent SDK** or headless Claude Code in CI. Each phase gets its own session, system prompt, and skills | `claude-code-action` in GitHub Actions |
| Artifact storage | Repository folder `.pipeline/<task-id>/` on the PR branch, archived with the PR | — |
| Artifact structure checks | JSON Schema for front matter; markdownlint with custom rules for required sections | Custom Python/TypeScript validator |
| QA Plan freeze | Committed content hash of the frozen spec; a CI check fails if the spec changes without a recorded Design re-approval | — |
| Tier escalation | Script that applies the domain/risk map to the changed paths and fails if the declared tier is too low | — |

## 2. Enforcement (quality gates)

Covers overview §3, §4, and §5.

| Mechanism | Tooling |
|---|---|
| Required checks and approvals | GitHub **Rulesets** / branch protection with "require review from code owners". GitLab: approval rules + protected branches |
| Domain and risk map | YAML map in the repo, applied as PR labels by `actions/labeler` or a script. Risk-area owners mirrored into **CODEOWNERS** so the platform enforces the right reviewers |
| Review policy (domain × risk → required approvals and tier) | **OPA / Conftest** evaluating PR metadata against policy-as-code |
| Gate integrity: new suppressions, coverage/mutation exclusions, skipped or deleted tests, gate config changed alongside code | **Danger** (danger.js) rules on the PR diff; CODEOWNERS on `.github/`, `gates.yaml`, the domain/risk map, and frozen specs |
| Contract conformance (Q-Check) | `oasdiff` (OpenAPI), `buf breaking` (protobuf), **Pact** (consumer-driven contracts) |
| Migration safety (expand/contract) | **Atlas** migration lint, or **Squawk** for PostgreSQL |
| Mutation testing (incremental on PR, full nightly) | **Stryker** (JS/TS, C#), **PIT** (Java/Kotlin), **mutmut** (Python), **cargo-mutants** (Rust), **gremlins** (Go) |
| CRAP score with legacy ratchet | **SonarQube** quality gate on new code (built-in "new code" ratchet), plus a script that computes CRAP from complexity (`lizard`) and coverage reports |
| SAST | **Semgrep** or **CodeQL** |
| Secrets | **gitleaks** |
| Dependencies | **OSV-Scanner** or **Trivy** for vulnerabilities; **Renovate** for upgrades, configured to map patch/minor/major to Tier 0/1/2 |
| DAST | **ZAP** against the preview environment; **Nuclei** for templated checks |
| Flaky test quarantine | Trunk Flaky Tests or Datadog Test Optimization; or a homegrown quarantine list driven by JUnit reports |

## 3. Environments and release

Covers overview Phase 8 (Deploy & Test) and Phase 9 (Release).

| Need | Tooling |
|---|---|
| Preview environment per PR | Kubernetes + Argo CD ApplicationSet (PR generator), or vcluster; Testcontainers for local runs |
| Feature flags | **OpenFeature** SDK, with Unleash or Flagsmith (self-hosted) or LaunchDarkly (SaaS) as the provider |
| Canary rollout with automated rollback | **Argo Rollouts** or **Flagger**, with analysis on Prometheus metrics (error rate, latency, business KPIs) |

## 4. AI judges and the Knowledge Base

Covers overview §3.2 (AI Judges), §4 (Architecture Judge), and the Artifact Lifecycle in §2.

| Need | Tooling |
|---|---|
| Judge calibration against seeded defects | **promptfoo** or **Inspect**, run nightly; results stored per judge (recall, false positive rate) |
| Judge model separation | Judge runs on a model family different from the generator, called through its own API |
| LLM tracing and cost | **Langfuse** (self-hosted): prompts, tool calls, token cost per task and per phase |
| Knowledge Base storage | Markdown docs-as-code in the repo with `owner` and `last_verified` front matter |
| Knowledge Base staleness | Scheduled CI job that flags pages past the verification period |
| Knowledge Base retrieval for agents | MCP server backed by pgvector or a hosted search service |

## 5. Agent security

Covers overview §6.

| Control | Tooling |
|---|---|
| Least privilege per phase | One **GitHub App** per phase with fine-grained permissions (Research read-only; Code writes to its branch only; no agent can merge, deploy to production, or edit gate config) |
| Short-lived credentials | **OIDC** federation to cloud providers; **HashiCorp Vault** for test-environment secrets |
| Sandboxing | Ephemeral containers or microVMs (gVisor, Firecracker, or E2B-style sandboxes) with an egress allowlist; **StepSecurity Harden-Runner** on GitHub Actions |
| In-agent tool restrictions | Claude Code permission settings and hooks |
| Audit trail | OpenTelemetry export from the agent runtime, traces tagged with task ID and PR number |

## 6. Monitoring and the feedback loop

Covers overview §5 (Feedback Loop via Delivery Metrics) and the sampled audit.

| Signal | Source → storage → view |
|---|---|
| DORA metrics (deployment frequency, lead time, change fail rate, recovery time) | **Apache DevLake** (open source, built-in DORA dashboards) collecting from Git, CI, deployments, and the incident tool. SaaS alternatives: Swarmia, LinearB, DX |
| Pipeline metrics (first-attempt gate pass rate, backtracks per task, human hours per phase, token cost) | Phase-transition events from the orchestrator → **OpenTelemetry** → Prometheus or ClickHouse → **Grafana** |
| Judge precision and recall | Nightly calibration results → Grafana |
| Incident attribution to domain and risk tag | **incident.io** or PagerDuty, linked to the deployment and PR; joined in DevLake |
| Automatic tightening | Grafana alert on change fail rate over a rolling window triggers a job that opens a PR raising thresholds in `gates.yaml` |
| Human-only loosening | Any PR that lowers a value in `gates.yaml` requires tech lead approval (CODEOWNERS + Conftest rule) |
| Sampled audit of auto-approved PRs | Scheduled job selects a random sample of merged auto-approved PRs and opens review issues assigned round-robin |

## Rollout

### Pilot

- GitHub Actions + labels as the state machine
- CODEOWNERS, Rulesets, Danger, Conftest
- Semgrep, gitleaks, SonarQube, mutation tool for the pilot language
- Headless Claude Code as the agent runtime
- Langfuse for agent traces and cost
- DevLake + Grafana for the DORA baseline (start before the pilot, per overview Stage 0)

### Scale-up

- Temporal for orchestration once backtracking and approvals outgrow labels
- Preview environments, Argo Rollouts, feature flags
- Judge calibration harness; judges switched from advisory to blocking
- One GitHub App per phase; sandboxed runners with egress control

## Open Questions

These determine the concrete choices above:

1. Git platform: GitHub, GitLab, or other?
2. Primary languages and runtimes (decides the mutation testing, complexity, and contract tools)?
3. Infrastructure: Kubernetes, serverless, or VMs, and which cloud (decides preview environments and progressive delivery)?
4. Preference for open source / self-hosted versus SaaS (DevLake vs Swarmia/LinearB, Unleash vs LaunchDarkly, self-hosted vs cloud Langfuse)?
