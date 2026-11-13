A Guide to Designing and Implementing an AI-Native SDLC: From Code Generation to Governed Architecture

1. Introduction and the Paradigm Shift in Software Development

Modern software engineering is undergoing a fundamental shift. With the arrival of generative neural models and autonomous AI agents, writing source code is no longer the bottleneck of development. The ability to generate thousands of lines of code instantly has created an illusion of enormous acceleration, yet in practice the delivery speed of finished products has not grown proportionally. On the contrary, teams that have not adapted their processes see more production incidents, architectural degradation, and an overall slowdown caused by the difficulty of maintenance.

Experience with tools for automated codebase generation (including popular spec-driven approaches and frameworks such as B-MAD) shows that documentation and code produced by agents "in one click" quickly become outdated and begin to degrade the quality of the agents that work on them next.

The main danger of unbalanced use of AI tools is that it turns the engineer into a so-called "Meat Proxy". In this negative model, the developer becomes a passive link between autonomous agents, reduced to mindlessly pressing Enter and forwarding tickets.

Definition of the "Meat Proxy" phenomenon: A state of the engineering process in which the developer loses understanding of how the system works, delegating the generation of architecture, code, tests, and documentation to agents, while acting as a formal mover of tasks without substantive validation.

The Problem of Declining Review Quality and Growing Code Volume

The limits of human attention are becoming the key factor constraining productivity:

* Reading speed limits: The speed at which a human can read code carefully remains physiologically limited. The SmartBear study of code review at Cisco found that reviewers going slower than about 400 lines per hour found defects at an above-average rate, while above about 450 lines per hour defect density fell below average in 87% of cases. Review effectiveness also drops after 60–90 minutes of continuous reading. A 5,000-line pull request therefore needs at least 12–13 hours of careful reading, split across many sessions.
* Growing change size: Because generation is cheap, the amount of code per pull request grows, making it physically impossible for the team to read all the generated code.
* Review shifting to automation: An increasing share of changes passes through automated reviewer agents rather than a human reader. Without an explicit policy for where human attention goes, it ends up spread thinly and randomly.

The Cascade of Errors Principle

The main economic and technical risk of using agents without a rigid process structure lies in the cascading of errors.

Stage where the error originates	Impact on subsequent stages	Economic and technical consequences
Early stages (Inreach / Design)	An error in the concept or contracts lands in the agents' first context windows.	Generation of faulty specifications, wrong implementation plans, and defective code.
Middle stages (Plan / Code Generation)	The agent generates code that looks functional but implements a false architectural model.	Large, wasted token spend and repeated fix cycles.
Late stages (Production)	Defects surface under high load or during integration.	Costly production incidents, loss of user trust, and erosion of the codebase.

To prevent cascading errors, the bottleneck of the SDLC moves from writing code to filling context, architectural design, deterministic control, and quality management at every transition.

2. Architecture of a Multi-Phase Feature Pipeline

Trying to hand an AI agent a complex task within a single context window inevitably leads to "Context Rot". The more information is passed in and the longer the reasoning trajectory within one window, the more the quality of the model's answers degrades.

To solve this, the feature development lifecycle is split into isolated phases, where each phase runs in its own context window with a specialized system prompt and produces a strict artifact. The full flow is: context filling (Inreach) → Research → Design → QA Plan → implementation Plan → Code → Contract Conformance Check (Q-Check) → Deploy & Test → Release.

Pipeline Tiers

Running every change through every phase makes small changes expensive, and teams will bypass a process that is too heavy for the task. The pipeline is therefore tiered. The tier is proposed by the engineer at the start of the task and enforced by a deterministic gate: if the changed paths or risk tags (see Section 5) require a higher tier, the gate escalates the task and the lower tier cannot be chosen.

Tier	Typical change	Phases
Tier 0 — Trivial	Typo, config value, copy change, dependency patch bump with no API change; no risk tags touched.	Code → deterministic gates → Release.
Tier 1 — Standard	Bug fix or small change inside one module, no contract change, no risk tags touched.	Inreach (short) → Research → Plan → Code → Q-Check → Deploy & Test → Release.
Tier 2 — Full	New feature, contract or schema change, cross-module change, any Core Domain change, or any risk tag touched.	All phases, with human approval at Inreach, Design, and Plan.

Phase 1: Inreach (Transferring engineering and business context)

This stage extracts tacit knowledge, business intent, architectural constraints, and pitfalls before any automated action begins.

* Mechanics: The GrillMe dialogue pattern. The AI agent interviews the developer, asking clarifying questions about business requirements, existing constraints, and the target behavior of the system. Questions the engineer cannot answer about business intent are routed to the product owner rather than guessed; the brief records who answered each one.
* Goal: Prevent the agent from filling gaps in the requirements with its own assumptions, so that the system is built the way the business needs rather than the way the agent happened to understand it.
* Artifact: Inreach Context Brief, including an explicit list of open questions and assumptions.

Phase 2: Research (Locating the knowledge base and code)

A targeted study of the codebase and the project knowledge base, based on the prepared context.

* Mechanics: Semantic and graph search across repositories and documentation to isolate only the relevant files, modules, and interfaces.
* Goal: Minimize the context window size for subsequent stages. Loading the entire codebase into the context window not only raises token costs but also worsens the model's output quality.
* Artifact: A research report with a precise map of affected files and architectural relationships (Research Map). The map also lists the domain and risk tags of the affected paths, which confirms or escalates the tier.

Phase 3: Design (High-level architectural design)

Creating formal architectural diagrams and specifications before a single line of code is written.

* Mechanics: Generating C4 structural models, Sequence Diagrams, and Data Flow Diagrams. The spec must include a "Considered alternatives" section listing at least one rejected approach and why it was rejected.
* Goal: Build a clear picture of how the new functionality fits into the existing system. This is the stage where fundamental architectural mismatches are uncovered.
* Human approval: To prevent rubber-stamping, the approver records in their own words the two or three main risks of the design and confirms or changes the chosen alternative. An approval without this note does not pass the gate.
* Artifact: System Design Spec.

Phase 4: QA Plan (Integration testing specification)

Defining the feature's testing strategy before the implementation plan is drawn up.

* Mechanics: Defining interaction contracts and designing end-to-end and integration ("black box") test scenarios.
* Goal: Pin down the module's expected external behavior independently of its internal implementation.
* Freezing: Once the QA Plan passes its gate, the contracts and scenarios are frozen (stored with a content hash). Later phases may add tests but may not weaken, remove, or rewrite the frozen ones. Changing a frozen contract is a design change and goes back to Phase 3.
* Artifact: QA Test Specification (test plan and contract specification).

Phase 5: Plan (Implementation breakdown)

Breaking the architectural design into a sequence of small, isomorphic, and independently verifiable steps.

* Mechanics: Splitting the task into logical layers (for example, domain -> use cases -> interface adapters).
* Goal: Create isolated implementation phases, each of which can be generated and tested independently. The plan should not contain large chunks of code; its job is decomposition.
* Human approval (Tier 2): A short review that the steps cover the whole design and nothing outside it. The plan is a small artifact, so this costs minutes, and a decomposition error caught here is far cheaper than one caught in code.
* Artifact: Implementation Roadmap.

Phase 6: Code (Step-by-step code generation)

Execution of the plan by autonomous code-generation agents.

* Mechanics: Iterative generation of source code and unit tests for each item of the implementation plan.
* Goal: Produce clean, strictly typed code that follows the project's local rules.
* Artifact: A set of source files and local unit tests.

Phase 7: Q-Check (Contract Conformance Check)

Verifying that the written code satisfies the frozen contracts and test plan. The direction of this check is one-way: the code must conform to the specification, never the other way round.

* Mechanics: Running the frozen QA scenarios against the implementation and diffing the implemented contracts (API schemas, events, DB schema) against the frozen ones.
* Goal: Detect any divergence between code and specification. A divergence is never resolved by editing tests to match the code. Either the code is fixed, or — if the specification turned out to be wrong — the task backtracks to Design (see Backtracking Rules).
* Artifact: A conformance report and the integration test suite (the frozen scenarios plus any added tests).

Phase 8: Deploy & Test (Automated deployment and test run)

Deploying the changes to an isolated test environment and running the verification scenarios.

* Mechanics: Automated deployment to a feature environment (or a local container) and a run of the full integration test suite.
* Goal: Confirm that the feature works in an environment close to production.
* Artifact: A green integration test run report.

Phase 9: Release (Controlled rollout)

A green test environment does not prove the change is safe in production. Release is part of the pipeline, not an afterthought.

* Mechanics: New behavior ships behind a feature flag or via a canary rollout. Rollout proceeds in stages with automated health checks (error rate, latency, key business metrics) and an automated rollback trigger. Database migrations must be backward compatible with the previous release (expand/contract pattern).
* Goal: Limit the blast radius of defects the earlier gates missed and make recovery a routine action rather than an incident.
* Artifact: Rollout record (flag state, stages, health check results).

Backtracking Rules

The pipeline is not one-way. When a later phase discovers that an earlier artifact is wrong, the task returns to the phase that owns that artifact, and every artifact downstream of it is regenerated.

Discovery	Return to	Human involvement
A step in the plan cannot be implemented as written.	Plan	Plan re-approval (Tier 2).
A frozen contract or test scenario needs to change.	Design	Design re-approval; QA Plan refrozen.
The design does not fit the existing system.	Research or Design	Design re-approval.
A business requirement was misunderstood.	Inreach	Engineer and product owner.
The same gate fails after the retry limit (for example, 3 attempts).	—	Escalated to a human; the agent stops.

Artifact Lifecycle

The pipeline itself generates a large amount of documentation, which is exactly what the introduction warns goes stale. Artifacts therefore have an explicit lifecycle:

* Per-task artifacts (Context Brief, Research Map, Design Spec, QA Spec, Roadmap, reports) live with the pull request and are archived with it. They are not loaded into future agents' context by default.
* At merge, a distillation step extracts what has lasting value: architectural decisions become ADRs, new or changed patterns become Knowledge Base updates. This step is a proposal that the Knowledge Base area owner approves.
* The Knowledge Base has a named owner per area, and every page has a "last verified" date. Pages not verified within a set period (for example, 6 months) are flagged, and the architecture judge treats them as lower-confidence.

3. The Quality Gates System

Transitions between pipeline phases cannot happen automatically without passing checkpoints — Quality Gates. A quality gate is a barrier that evaluates the current phase's artifact against established standards.

Gates go through three levels of verification in sequence: deterministic checks in code, evaluation by an AI Judge agent, and expert validation by a human.

1. Deterministic Code Gates

The first and mandatory line of defense, which consumes no API tokens and runs on classic scripts and utilities.

* Artifact structure checks: Automated validation that all required sections are present in Markdown specification files, and validation of JSON/YAML schema formats.
* Tooling checks: Running static type checkers, linters, and code formatters.
* Test execution: Guaranteed execution of the full test suite. This gate prevents a common agent behavior: avoiding running tests, or reporting success without actually running them.
* Gate integrity: Agents optimize for passing the gate, so the gates themselves must be protected. Gate configuration, thresholds, CI pipeline definitions, the domain/risk map, and frozen QA specs are protected by CODEOWNERS and cannot be changed in the same pull request as the code they check. The gate also fails on new lint/type suppressions, coverage or mutation exclusions, and skipped or deleted tests unless a human explicitly approves them.
* Flaky tests: A test that fails intermittently is quarantined and assigned an owner, rather than retried until green. Retries hide real defects and make the agent's retry loop unreliable.

2. AI Judges / Reviewers

Automated neural arbiters that check the substantive quality of artifacts.

* Separation of model families: The judge agent should use a different family of language models than the one that generated the artifact. This reduces, but does not remove, shared blind spots: models from different families are trained on overlapping data and still make correlated mistakes. A different family is a precaution, not a guarantee.
* Scope of validation: Checking that the system prompt's conditions are fully met, detecting logical contradictions in the design, and verifying adherence to knowledge base conventions.
* Calibration: A judge whose accuracy is unknown is not a gate. Each judge is measured regularly against a benchmark of artifacts with seeded, known defects, and its findings are compared with defects later found by humans and in production. A judge starts in advisory mode and becomes blocking only after its measured recall and false positive rate meet the agreed target.

3. Human Oversight: Human-in-the-Loop vs Human-on-the-Loop

Human involvement in the pipeline is strictly differentiated to balance speed and reliability:

* Human-in-the-Loop (mandatory direct involvement): The engineer is involved in the Inreach phase (context transfer, with the product owner for business questions), approves the Design (with the written risk note described in Phase 3), and, for Tier 2, approves the Plan. These are the phases where an error cascades through everything downstream.
* Human-on-the-Loop (observation mode): In the Research, QA Plan, Code, Q-Check, Deploy, and Release phases, the engineer acts as an observer. The system executes the steps automatically and stops for a human on fatal failures, on any backtrack to Design or Inreach, or when a gate still fails after the retry limit.

4. Deep Code Validation in the CI/CD Pipeline

Code generation by neural networks creates a specific risk: agents can tailor the implementation to pass shallow tests, or generate synthetic tests that effectively test themselves. To keep the codebase quality high, rigorous tool-based validation methods are applied.

Mutation Testing

Classic unit testing only shows that lines of code were executed, not how good the checks themselves are. Mutation testing evaluates the robustness of test coverage as follows:

1. A specialized framework creates modified copies of the program ("mutants") by changing logical operators (for example, replacing > with <=), boundary conditions, and return values.
2. The existing test suite is run against each mutant.
3. If the tests keep passing on the broken code ("the mutant survived"), this indicates that the tests are weak or do not actually check behavior.
4. The pipeline blocks the build until the Mutation Score exceeds the target threshold.

Running the full mutation suite on every pull request is too slow for most codebases. On pull requests, mutants are generated only for changed lines (incremental mutation testing); the full run happens nightly and its results feed the DORA review.

Composite Code Metrics and the CRAP Score

To automatically assess the risk of changes, the CRAP (Change Risk Anti-Patterns) metric is used, which combines cyclomatic complexity and test coverage:

\text{CRAP}(m) = C^2 \cdot (1 - p)^3 + C

where C is the method's cyclomatic complexity and p is the fraction of the method covered by automated tests.

Because complexity enters the formula squared, the score climbs steeply for complex methods with incomplete coverage. If the metric exceeds a threshold (for example, CRAP > 30), the automated build pipeline rejects the code and requires either simplifying the method's design or covering it with integration and mutation tests. The threshold applies to new and changed methods; existing legacy code is handled with a ratchet (its score may not get worse), so adopting the gate does not block all work on old code.

Static and Dynamic Analysis (SAST / DAST)

Before code is handed to an engineer for review, deterministic security and reliability checks are run:

* SAST (Static Application Security Testing): Scanning source code for vulnerabilities, memory leaks, hardcoded secrets, and suboptimal database access patterns. Runs on every pull request.
* DAST (Dynamic Application Security Testing): Automated testing of the application deployed to the feature environment for resilience against attacks (SQLi, XSS, access control violations). Runs on every pull request that touches a risk tag, and nightly for everything else.

The Architecture Judge Agent and the Team Knowledge Base

To check that the code complies with company standards, a specialized judge agent is used that retrieves relevant patterns from the engineering Knowledge Base (retrieval-augmented, not a separately trained model).

The judge agent compares the generated Pull Request with the patterns from the team's knowledge base and scores how well the architectural decisions match. The architecture agent's effectiveness depends directly on how up to date the company Knowledge Base is, which is why the Knowledge Base has owners and a verification cycle (see Artifact Lifecycle). If the knowledge base contains clearly described patterns (for example, Clean Architecture layering rules, logging and error handling standards), the agent can catch architectural deviations before the pull request is sent for human review.

5. A Domain- and Risk-Based Pull Request Review Strategy

Attempting detailed human review of 100% of AI-generated code inevitably paralyzes development. To allocate engineers' attention rationally, review depth is decided along two independent axes: the DDD domain of the change (how much business value is at stake) and its risk tags (how much damage a defect can do). Domain alone is not enough: authentication is "Generic" in DDD terms, but a defect there is among the most damaging possible.

The Domain and Risk Map

* Tech leads maintain a map from code paths to a domain (Core, Supporting, Generic) and to risk tags, in a CODEOWNERS-style file in the repository.
* A deterministic gate applies the map to every pull request. Agents cannot change the map; changes to it require tech lead approval.
* A pull request that touches several domains is reviewed under the strictest one that applies.
* Paths missing from the map are treated as Core until classified.

Risk tags (initial set): authentication and authorization, payments and money movement, personal or sensitive data, database schema migrations and data backfills, public or external API contracts, infrastructure as code and CI/CD configuration, cryptography and secrets handling, quality gate configuration.

Review Rules by Domain Type

1. Core Domain:
  * Essence: The product's unique logic that creates the business's main competitive value.
  * Rules: 100% mandatory detailed human review. The pull request is read by at least two experienced engineers. Reviewers check not only tests and correctness, but also the code's room to evolve, performance, and the rigor of the domain model.
2. Supporting Domain:
  * Essence: Business-adjacent functionality needed for the core domain to work, but not unique.
  * Rules: Simplified control. Review by one colleague (not the task author) as a structured look at the changes, provided all automated gates are green.
3. Generic Domain:
  * Essence: Standard technical functionality (notifications, internal tooling, infrastructure plumbing without risk tags).
  * Rules: Automatic approval. If deterministic checks, SAST/DAST, mutation tests, and the architecture AI judge all pass, the pull request is accepted without mandatory line-by-line human reading.

Risk tag override: Any pull request that touches a risk tag gets at least one human review by an owner of that risk area, regardless of domain. Core changes that touch a risk tag need one of their two reviewers to be that owner.

Sampled audit of auto-approved changes: A random sample of auto-approved and simplified-review pull requests (for example, 5–10%) is read in full by a human after merge. This measures how well the automated gates and the judge are actually working, and keeps engineers familiar with code they would otherwise never read.

Keeping Human Understanding of the System

Code that no human reads becomes code no human understands, and incidents in it take longer to resolve. To avoid a slow return of the "Meat Proxy" problem:

* Each domain has a named owning engineer who is expected to understand it, whether or not every change was reviewed line by line.
* The sampled audit rotates across engineers.
* Incident reviews include the question "did anyone on the team understand the failing code before the incident?" and feed back into the domain map.

Feedback Loop via Delivery Metrics

The strictness of Quality Gates and the review strategy are not static. They are adjusted based on the four DORA metrics, which cover both speed and stability:

* Deployment Frequency: How often changes reach production.
* Change Lead Time: Time from commit to running in production.
* Change Fail Rate (CFR): The share of deployments that lead to a failure in production.
* Failed Deployment Recovery Time (MTTR): The time to restore service after a failed deployment.

Speed metrics matter as much as stability: the whole point of the pipeline is faster delivery without lower quality, and stability metrics alone can only ever argue for more strictness.

The pipeline is also measured directly: gate pass rate on the first attempt, number of backtracks per task, token cost per task, human hours per phase, and judge precision and recall. These show which phases earn their cost.

How adjustments are made:

* Tightening is automatic. When CFR rises in a service or domain over a rolling window (for example, 30 days, with a minimum number of deployments so that single incidents don't trigger it), the system raises the minimum mutation score, lowers CRAP tolerances, and moves Supporting domains to mandatory human review.
* Loosening is a human decision. Restoring or relaxing thresholds requires an explicit tech lead decision, recorded with the supporting data.
* Each incident is attributed to the domain and risk tags of the change that caused it, so that the per-domain numbers mean something.

6. Agent Security

Agents with write access to code and CI are a new attack surface and need the same controls as any other privileged actor.

* Least privilege: Each phase's agent gets only the permissions that phase needs. Research is read-only; only Code can write to the working branch; no agent can merge, deploy to production, or edit gate configuration.
* Secrets: Agents never receive production credentials. Test environments use dedicated, scoped, short-lived credentials.
* Prompt injection: Repository content, tickets, documentation, and dependency code are untrusted input that may contain instructions. Agents run in a sandbox with restricted network access, and actions with external effects require a gate or a human.
* Audit trail: Every agent action (prompts, tool calls, produced artifacts) is logged and linked to the pull request.

7. Work That Is Not a New Feature

Not every change is a feature. Each work type maps onto the tiers:

* Bug fix: Tier 1 by default; Tier 2 if the fix changes a contract or touches a risk tag. The first step is a failing test that reproduces the bug.
* Production hotfix: An expedited path — Code, deterministic gates, and one human review — with a mandatory follow-up task within an agreed time that runs the normal tier and adds the missing tests.
* Refactoring: Tier 1 when behavior is unchanged; the existing tests are the frozen contract and may not be modified in the same pull request.
* Dependency upgrades: Tier 0 for patch versions; Tier 1 for minor versions; Tier 2 for major versions or anything touching a risk tag.

8. AI-SDLC Adoption Checklist for a Technical Leader

Transforming engineering processes requires adopting practices step by step, from the foundation up to automation, and measuring whether each step helps.

Stage 0: Baseline and Pilot

1. Measure the current DORA metrics and review load for at least one month before changing anything, so improvement can be shown.
2. Choose one pilot team and one service with a mix of domains.
3. Agree on success criteria (for example, lead time improves while CFR does not worsen) and rollback criteria before the pilot starts.

Stage 1: Preparing the Foundation and the Knowledge Base

1. Create the company Knowledge Base: document architectural patterns, layering rules, and coding standards, with a named owner for each area and a verification cycle.
2. Introduce the GrillMe dialogue pattern for defining and refining tasks, with a path for routing business questions to the product owner.
3. Establish the rule that high-level architectural specifications (Design Spec) must be created and approved before coding starts on Tier 2 tasks.
4. Create the domain and risk map and protect it with CODEOWNERS.

Stage 2: Setting Up Automated Quality Gates

1. Set up deterministic artifact structure checks at every phase of the pipeline.
2. Set up gate integrity checks: protected gate configuration, and detection of suppressions, exclusions, and skipped tests.
3. Add incremental mutation testing to pull requests and a full nightly run.
4. Deploy CRAP metric calculation (with a ratchet for legacy code) and SAST with strict build blocking when thresholds are exceeded.
5. Set up feature flags, staged rollout, and automated rollback for Release.

Stage 3: Introducing AI Judges and Domain-Based Review

1. Set up a judge agent on a model from a different family, run it in advisory mode, and build the seeded-defect benchmark to calibrate it before making it blocking.
2. Approve the review rules: domain × risk routing, 100% deep human review for the Core Domain, risk tag override, and automated acceptance for Generic.
3. Start the sampled audit of auto-approved changes.
4. Establish continuous monitoring of all four DORA metrics plus pipeline metrics, with automatic tightening and human-approved loosening.

9. Open Decisions

These values are starting points that each team must set and then calibrate against its own data:

* The exact boundaries between Tier 0, 1, and 2.
* The retry limit before a failing gate escalates to a human.
* Mutation score and CRAP thresholds per domain.
* The sample rate for the audit of auto-approved changes.
* The recall and false positive targets a judge must meet before it becomes blocking.
* The rolling window and minimum deployment count for automatic tightening.
* The Knowledge Base verification period.
* The full list of risk tags for the specific product.
