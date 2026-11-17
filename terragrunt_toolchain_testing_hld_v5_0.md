# Terragrunt Toolchain Automated Testing - High-Level Design

**Version:** 5.0  
**Status:** Proposed for architecture review  
**Scope:** Phase 1 candidate validation, AWS Test execution, evidence, and controlled release; Phase 2 direction  
**Baseline:** Terragrunt Toolchain Automated Testing HLD v4.1  
**Primary design objective:** Establish a reusable testing mechanism that can execute future team-authored tests against a newly proposed Toolchain revision, without redesigning the CI/CD architecture for each test.

## 1. Overview

The solution validates a proposed **Catalog Toolchain revision before it is released for consumption**. A Catalog pull request (PR) triggers layered checks, deploys that exact candidate revision through Workshop into a dedicated four-account AWS Test organization, runs selected tests against the deployed Toolchain, retains evidence, cleans up candidate resources, and returns an authoritative GitHub quality-gate result.

The system validates both the infrastructure definition and the behavior of the Toolchain **as a consumer would use it**. It must therefore support tests executed centrally through AWS APIs and tests executed **through the actual Toolchain runners** deployed in the Test accounts. A test author's choice of execution context must not require a new orchestration architecture.

This HLD defines architectural responsibilities, stage boundaries, test selection, execution targets, candidate provenance, security boundaries, lifecycle, evidence, and release controls. It deliberately does not prescribe individual test assertions, buildspec commands, IAM policy JSON, file layouts, or Terraform/Terragrunt implementation details.

### 1.1. Design principles

1. **Test the candidate, not the last release.** The deployed revision must be the same proposed Toolchain revision being approved.
2. **Keep one orchestrator.** GitHub Actions controls the full PR validation state; Workshop owns deployment and an approved AWS execution path runs the selected tests.
3. **Layer tests by purpose, select by policy.** Testing layers explain coverage; mandatory and change-based rules determine which tests run.
4. **Use real identities for behavioral assertions.** A Dev runner's permissions cannot be validated by a privileged Toolchain executor impersonating a different principal.
5. **Prefer a small reusable execution platform.** New tests extend the Catalog-owned registry and test implementations rather than creating a pipeline or CodeBuild project for each test.
6. **Fail closed and preserve evidence.** Missing results, identity mismatch, failed cleanup, and unverified releases cannot be reported as successful validation.
7. **Separate architecture from implementation.** The HLD specifies contracts and invariants; the LLD specifies commands, policies, and resource definitions.

## 2. Scope and Requirements

### 2.1. In scope: Phase 1

- Validation of Catalog PRs targeting `main`, including PR updates, before merge.
- Sequential, fail-fast validation layers with appropriate result aggregation inside each layer.
- Mandatory baseline tests and rule-selected tests based on Catalog changes.
- Candidate Plan and Apply using the existing Workshop deployment mechanism and the `AWS/org-test` hierarchy.
- A dedicated AWS Test organization with Toolchain, Dev, Staging, and Prod accounts.
- AWS integration tests, including API-based and runner-context checks where needed, plus limited customer-like E2E tests. The integration execution mechanism remains open.
- Hybrid environment lifecycle: persistent foundation and temporary candidate Toolchain resources.
- Structured evidence, actionable debugging, cleanup, and enforced Catalog release gates.
- One active AWS candidate validation at a time against the shared Test environment.

### 2.2. Out of scope: Phase 1

- Comprehensive production-workload compatibility certification.
- Dedicated upgrade, idempotency, and destroy **test scenarios**. Operational teardown is nevertheless mandatory.
- Dependency-graph-based Test Impact Analysis (TIA), dynamic risk scoring, and a separate test-selection service.
- Replacing Workshop's established deployment engine or Runtime's deployment architecture.
- Building a new general-purpose workflow platform, scheduler, or test portal.
- Detailed test inventory, test fixtures, code, IAM statements, buildspecs, and exact cleanup commands.

### 2.3. Success criteria

The mechanism is successful when an engineer can add a test in Catalog, declare when and where it runs, and have it executed with reproducible evidence during the appropriate Catalog PR runs. A failed mandatory check, AWS test, candidate deployment, or required cleanup must prevent the PR from being marked fully validated. The released revision must be traceable to the approved validation evidence.

## 3. Solution

### 3.1. Testing Strategy and Layers

Testing layers define **what is being validated** and the logical order. They are distinct from test selection and execution technology. The team may author tests using suitable frameworks, provided they satisfy the common test contract described in Section 3.5.

| Ordered layer                         | Validation objective                                                                  | Execution boundary                          | Selection approach                                    | Gate                      |
| ------------------------------------- | ------------------------------------------------------------------------------------- | ------------------------------------------- | ----------------------------------------------------- | ------------------------- |
| **1. Static Validation**              | Syntax, formatting, Terragrunt/Terraform structure, offline configuration consistency | GitHub, no AWS access                       | Changed validation units with full dependency context | Required                  |
| **2. IaC Security**                   | Infrastructure configuration risks and unsafe defaults                                | GitHub, no AWS access                       | Affected units                                        | Required                  |
| **3. Policy Compliance**              | Agreed Toolchain/IaC policies and guardrails                                          | GitHub, no AWS access                       | Affected units                                        | Required                  |
| **4. Native / Module Validation**     | Offline module contracts and assertions where supported                               | GitHub, no AWS access                       | Affected modules                                      | Required where applicable |
| **5. Test Selection**                 | Produce a complete, valid plan of AWS tests                                           | GitHub                                      | Mandatory baseline plus change-based rules            | Required                  |
| **6. Candidate Plan**                 | Candidate produces a valid plan against representative Test stacks                    | Existing Workshop flow, Test organization   | Defined Test deployment scope                         | Required                  |
| **7. Candidate Apply**                | Candidate can provision the required Toolchain and fixtures                           | Existing Workshop flow, Test organization   | Defined Test deployment scope                         | Required                  |
| **8. AWS Integration**                | Deployed resources, IAM, APIs, and cross-account behavior operate correctly           | AWS Test organization; execution path TBD   | Mandatory plus rule-selected                          | Required                  |
| **9. Limited E2E**                    | A customer-like workflow operates through the deployed Toolchain                      | Golden Path repository using Test Toolchain | Mandatory plus rule-selected                          | Required                  |
| **10. Evidence, Cleanup, Final Gate** | Results are complete, candidate is removed, approval is trustworthy                   | GitHub and AWS                              | All required stages                                   | Required                  |

**Stage semantics.** A failed dry-check layer prevents AWS deployment. A failed Plan prevents Apply; a failed Apply prevents behavioral tests; a failed Integration stage prevents E2E. Independent tests already running within a stage should complete and report where safe. Evidence collection and cleanup are unconditional recovery obligations, not downstream validation tests that can be skipped after failure.

The exact offline tools and their order must be aligned with the existing Catalog workflow during implementation. AWS-independent tests must not silently use live providers, remote AWS state, or credentials; tests requiring AWS belong in the AWS execution stages.

#### 3.1.1. Coverage boundaries

The Phase 1 baseline should provide confidence in five broad areas, rather than enumerate every possible test:

| Coverage dimension | Intended confidence |
|---|---|
| **Configuration correctness** | Proposed Toolchain code is structurally valid and meets defined offline contracts |
| **Security and policy** | IaC guardrails pass, and deployed identity/network boundaries behave as intended |
| **Deployment viability** | The proposed revision can Plan and Apply against representative Test account stacks |
| **Runtime integration** | Created Toolchain components and cross-account interactions operate as expected |
| **Consumer workflow** | A representative customer-like project can use the deployed Toolchain end to end |

This is a **representative release gate**, not an assertion that every possible client configuration is tested.

### 3.2. Test Selection and Extensibility

Phase 1 combines two independent selection mechanisms:

- **Mandatory baseline:** designated tests run for every candidate regardless of changed files.
- **Rule-selected coverage:** additional tests run when a Catalog change matches their declared applicability conditions.

Test authors define **when** a test should run as part of registering the test. Catalog remains the version-controlled source of truth for test implementations, test metadata, mandatory status, selection rules, execution context, stage, and expected result contract. A lightweight Catalog-owned selector interprets this information; there is **no separate selector service** in Phase 1.

#### 3.2.1. Selection sequence

1. GitHub identifies the candidate revision and the full PR changed-file set.
2. The selector loads the approved test registry and selection policy **from the candidate revision**, subject to independent policy validation.
3. It includes all mandatory tests and unions tests matched by change rules.
4. It validates test IDs, stage assignments, supported execution contexts, required account targets, and dependencies; duplicate selections are removed.
5. Changes to tests, registry, selection rules, or shared infrastructure are explicitly covered. Unknown or ambiguous impact produces a conservative broader suite or a failed selection gate, never an empty success by default.
6. It creates an immutable **Test Manifest** recording selected tests, reasons, execution contexts, target accounts, stages, and candidate identity.
7. GitHub retains the manifest and passes the relevant stage subset to the AWS Test Executor after deployment.

**Example:** A change to Toolchain IAM code selects the mandatory baseline and IAM-relevant integration tests. One selected test may run as an AWS API probe; another may request execution by the deployed Dev runner to verify an expected access denial. Both follow the same selection and result contract.

#### 3.2.2. Minimal test onboarding contract

A new test is added to Catalog with a small, reviewable definition. Its required fields are **test ID**, **testing layer**, **selection rule** (`always` or relevant changed paths/components), **execution target** (approved AWS context, including a candidate runner when needed), and **test entry point**. The test implementation defines its own assertions and expected outcome. Optional prerequisites, timeouts, and dependencies are specified only when needed. The registry/selector validates these definitions; a test cannot choose arbitrary AWS accounts or privileged roles.

The execution target is deliberately retained: selection answers **whether** a test runs, while the target answers **where and under which identity**. This is metadata, not a requirement to build a separate dispatch framework, matrix engine, or service. The final mechanism for running integration tests is not yet selected. Tests may use different approved frameworks while returning a common pass/fail result and diagnostic reference.

**Why this design:** A mandatory-only suite becomes slow as coverage grows; a change-only suite can miss systemic regressions; dependency-aware TIA requires an accurate graph and maintenance burden before there is enough empirical data. Mandatory plus explicit rules provides transparent, reviewable Phase 1 behavior and a straightforward migration path to TIA in Phase 2.

### 3.3. High-Level Architecture

```mermaid
flowchart TD
    PR[Catalog PR: candidate Toolchain revision] --> GH[GitHub Actions: orchestrator and quality gate]
    GH --> DRY[Offline validation and test selector]
    DRY --> MF[Immutable test manifest and candidate identity]
    GH --> WS[Workshop: existing deployment workflow]
    WS --> ORG[AWS Test organization / AWS-org-test]
    ORG --> TC[Test Toolchain account]
    ORG --> DEV[Test Dev account]
    ORG --> STG[Test Staging account]
    ORG --> PROD[Test Prod account]
    GH --> INT[Integration tests: execution approach open]
    MF --> INT
    INT --> AWS[Selected AWS behavior and runner-context checks]
    GH --> GP[Golden Path: existing customer CodePipeline]
    GP --> RUN[Actual candidate Toolchain runners]
    RUN --> DEV
    RUN --> STG
    RUN --> PROD
    INT --> EV[Structured results, logs, evidence archive]
    GP --> EV
    EV --> GH
    GH --> CL[Candidate cleanup and final status]
    CL --> REL[Validated merge / release-please / approved tag]
```

| Component                       | Responsibility                                                                                                                |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **Catalog**                     | Candidate Toolchain source, test code, test registry, selection policy, PR validation and release ownership                   |
| **GitHub Actions**              | Sole end-to-end orchestrator: sequencing, selection, invocation, correlation, waiting, evidence consolidation and PR status   |
| **Workshop**                    | Existing Plan/Apply/Destroy deployment capability; test-specific hierarchy under `AWS/org-test`                               |
| **Test Toolchain account**      | Shared Toolchain services and a possible central integration execution location; controlled test artifacts and evidence |
| **Test Dev / Staging / Prod**   | Representative customer account topology and candidate Toolchain deployment targets; Test Prod is not real production         |
| **Integration execution**       | Executes selected runtime checks through an approach still to be agreed; must preserve the required AWS identity and network context |
| **Candidate Toolchain runners** | Actual deployed runners used when a test must execute with consumer-equivalent identity, network, and permissions             |
| **Golden Path repository**      | Separate customer-like project/workflow used for selected E2E validation; not a second test orchestrator                      |
| **Runtime**                     | Consumes approved released versions; must not bypass the tested-version control                                               |

**Execution boundary.** The integration execution mechanism is not yet selected. A central CodeBuild executor with access to candidate runners is one option to discuss. Any solution should avoid creating a separate project per test and must preserve the required identity, security boundaries, and result contract.

### 3.4. End-to-End Candidate Validation Flow

```mermaid
flowchart TD
    A[1. Catalog PR opened or updated] --> B[2. Offline validation layers]
    B -->|Pass| C[3. Resolve mandatory and rule-selected tests]
    C -->|Valid manifest| D[4. Acquire Test environment lock and authorize execution]
    D --> E[5. Workshop Plan using candidate revision in org-test]
    E -->|Pass| F[6. Workshop Apply: deploy candidate Toolchain]
    F -->|Pass| G[7. Selected AWS integration tests: execution TBD]
    G -->|Pass| H[8. Selected Golden Path E2E]
    H --> I[9. Consolidate evidence]
    I --> J[10. Destroy temporary candidate and verify reset]
    J -->|Success| K[11. Final GitHub required check]
    K -->|Pass| L[12. Merge, release-please, version provenance gate]
    B -->|Fail| X[Record failure]
    C -->|Invalid| X
    E -->|Fail| X
    F -->|Fail| X
    G -->|Fail| X
    H -->|Fail| X
    X --> I
    J -->|Failure| Q[Quarantine Test environment; block next deployment and final approval]
```

**Flow clarifications:**

- **Steps 2 and 3 are different.** Step 2 evaluates the candidate's offline correctness. Step 3 determines the **AWS test plan** to execute later. Selection is not execution.
- **Steps 5 and 6 deploy the proposed version.** The candidate is the exact new code revision in the PR, not a previously published Toolchain tag. Workshop performs Plan and Apply with that revision in `AWS/org-test`.
- **Step 7 executes the selected integration tests.** The execution mechanism is still open. Tests that validate runner permissions or network behavior must use the actual runner context.
- **Step 8 uses the Golden Path repository as a consumer fixture.** It runs through the existing customer CodePipeline: Source and Build, then the configured Plan, Approval, and Deploy stages. The pipeline uses the actual Toolchain runners. How automated tests handle manual approvals remains to be defined.
- **Steps 9 and 10 must still run on failure.** Failure to complete candidate cleanup leaves the environment unavailable for another candidate until recovery.
- **Step 11 is the authoritative verdict.** A successful AWS API trigger or a green CodeBuild build without complete required test evidence is insufficient.

#### 3.4.1. Candidate revision and Workshop integration

Workshop currently consumes Toolchain versions through configuration such as `catalog_version` in `account.hcl`. For Test deployments, Workshop will support a **controlled candidate override** that points to the proposed Catalog revision. This override is allowed only under `AWS/org-test` and must not alter production deployment behavior under `AWS/org-prod`.

The candidate revision is represented by an immutable commit identity for the PR, but the design intent is straightforward: **Plan, Apply, and subsequent tests all validate the same new Toolchain version**. The workflow records the candidate identity and the effective deployed revision; a mismatch fails validation. The exact override syntax and artifact retrieval belong to the LLD.

For the shared Test environment, AWS candidate deployment is **serialized**. A run holds the environment lock through evidence capture, cleanup, and reset verification. Superseded or cancelled PR runs must not release the lock while leaving an active deployment behind.

### 3.5. AWS Test Execution Mechanism

The Integration Test execution mechanism is still an **open implementation question**. The testing stage remains part of Phase 1, but no execution approach has been selected.

One possible option is a central CodeBuild project that runs selected AWS tests and, when needed, triggers checks through the actual candidate runners. The current runners normally operate through CodePipeline and use defined buildspecs. Direct test execution, safe access to runner permissions, and result collection still need to be validated. This is an option for discussion, not an approved design.

#### 3.5.1. Executing selected tests

Integration tests cover runtime behavior not already proven by offline checks, Plan/Apply, or the Golden Path E2E. Some checks can run through scoped AWS APIs. Checks of a runner's permissions or network access must use that runner's real execution context.

The Golden Path is separate from the integration execution choice. It exercises a customer-like flow through the **existing Toolchain CodePipeline**, including Source, Build and the configured environment stages. The central Build produces a shared artifact for the environment stages; Plan and Deploy run in the relevant accounts. Manual approvals in automated test runs remain a pilot design question.

#### 3.5.2. Minimal execution and results contract

Regardless of the chosen execution mechanism, each selected test must be linked to the **candidate revision**, **validation run ID**, **test ID**, **execution target**, and **test entry point**. Each execution must return a clear **pass/fail/error/timeout result**, an execution reference, and diagnostic evidence.

GitHub checks that results cover all required tests. A missing result, mismatched runner identity, or untracked execution cannot be treated as a pass. Test code from an unreviewed Catalog PR must not control privileged roles, runner configuration, or unrestricted test invocation. Exact permissions, test delivery, and logging mechanisms will be decided with the execution approach.

### 3.6. AWS Test Environment and Lifecycle

#### 3.6.1. Four-account topology and prerequisites

| Account | Architectural purpose |
|---|---|
| Toolchain | Central Toolchain services, test coordination and evidence; possible location for integration execution |
| Dev | Represents the less-trusted customer development environment and its runner behavior |
| Staging | Represents the trusted staging environment and its runner behavior |
| Prod | Represents the trusted production-like environment **within the Test organization only** |

The Workshop Test hierarchy is maintained under `AWS/org-test`, distinct from `AWS/org-prod`. It can be adapted to the testing use case without modifying the production hierarchy. The baseline must supply the required account provisioning, administrative roles, regional enablement, network connectivity, and shared dependencies for representative deployment. Candidate-specific configuration, runners, and Toolchain stacks are created by Workshop.

The existing Toolchain deployment has prerequisites including LZA-provisioned accounts and admin roles, enabled target regions, spoke VPCs and runner subnets, egress for services such as CodeArtifact, and consumer configuration files. These inform the **readiness boundary**, but this HLD does not replicate a detailed organization setup runbook.

#### 3.6.2. Hybrid lifecycle

**Approved principle:** Persistent foundation plus full teardown of the temporary candidate deployment after each validation attempt.

- **Persistent foundation:** Account/organization setup, shared hub/spoke connectivity and other reusable prerequisites that should not be rebuilt for every PR.
- **Temporary candidate:** The Toolchain deployment and test-specific resources required to validate the new revision, including candidate runners and representative test fixtures as appropriate.
- **Lifecycle owner:** Workshop owns infrastructure operations; GitHub orchestrates invocation and observes completion. The selected test execution mechanism may participate in test-level teardown but does not own full infrastructure state.

A partial Apply, failed test, timeout, or cancelled run must trigger best-effort teardown, reconciliation, and a verified reset before the next candidate begins. If reset cannot be verified, the shared Test environment is **quarantined** and an owner is alerted. The original test failure and the cleanup failure are both preserved; cleanup failure cannot convert a failing run into success.

The exact permanent/temporary resource inventory, Terraform state boundaries, and whether specific shared Toolchain dependencies must remain persistent are **LLD and pilot decisions**, constrained by security and cost. The principle is fixed; the classification is not.

### 3.7. Quality Gates, Reporting, Evidence and Release

#### 3.7.1. Result propagation and debugging

GitHub is the PR-facing source of truth. It tracks each specific Workshop execution and the execution IDs of the selected test mechanism and Golden Path Pipeline until terminal state. The final required check includes selected test IDs and reasons, stage/test outcomes, candidate identity, target account and execution mode, failure summaries, and links to AWS diagnostics.

The chosen execution mechanism must produce structured test results (preferably JUnit XML or another supported format) and useful diagnostic logs. The existing customer Pipeline records execution status in the Toolchain account, while its CodeBuild jobs have logs in their respective accounts. Integration and Golden Path results must reach the common reporting contract rather than rely only on a build or pipeline exit status. An expected access denial must be recorded as **pass**, not treated as an execution failure.

**Fail-closed conditions:** An AWS build that starts but never finishes, a missing required report, incomplete test results, a mismatched candidate revision, an untracked runner job, a timeout, an unsafe target account, or a failed required cleanup cannot produce a passing final gate.

#### 3.7.2. Evidence retention policy

**Recommended HLD policy:** Retain an immutable release-validation record and its structured evidence for **at least 12 months**, subject to the organization's formal retention/security policy; retain detailed execution logs for **90 days in CloudWatch** and export/compress required diagnostic bundles to encrypted S3 for the same 12-month validation window. Use S3 lifecycle transitions for older evidence and automatic expiration when the approved period ends. Preserve security-incident or compliance-hold records according to the applicable hold policy.

This is a **proposed retention baseline for approval**, not a claim of an existing organizational requirement. The exact periods and storage classes should be confirmed with Security/Operations and validated against pilot costs. The choice deliberately distinguishes fast operational debugging from durable release provenance. AWS CodeBuild's native test-report console retention is only **30 days**, so durable evidence must be exported independently.

Evidence must include the Test Manifest, candidate and released revision identities, Plan/Apply references, structured per-test results, failure diagnostics, cleanup outcome, and final GitHub verdict. Credentials and secrets must not be archived.

#### 3.7.3. Release integrity

1. Required Catalog checks protect the feature PR merge into `main`.
2. `release-please` creates the release PR and, following approval, the version tag.
3. The release process must establish **provenance between the tested candidate and the release artifact/tag**. If release generation changes deployable Toolchain content, the release candidate requires equivalent validation before publishing or consumption.
4. Runtime must accept only an approved released Toolchain version through an enforceable consumption control, not merely through convention.

The exact provenance attestation, allowed-version enforcement and release PR validation mechanism are LLD/release-control design items. The HLD requirement is non-bypassable traceability between validated content and consumed content.

### 3.8. Phase 1 versus Phase 2

| Capability | Phase 1 | Phase 2 / possible evolution |
|---|---|---|
| Validation layers | Explicit ordered offline, deployment, integration, E2E stages | Extend coverage based on observed failures |
| Selection | Mandatory baseline plus simple change rules | Dependency-aware TIA and risk-based rules if justified |
| Test authoring | Catalog-owned test registry and minimal execution contract | Richer dependency metadata and self-service tooling |
| AWS execution | Selected Integration Tests; execution approach to be agreed and validated | Optimize execution based on pilot results |
| Candidate deployment | Workshop Plan/Apply in `AWS/org-test` | Broader representative environments or lifecycle scenarios |
| Environment lifecycle | Persistent foundation, ephemeral candidate, serialized AWS validation | Controlled parallel candidate environments if economically justified |
| Reporting | GitHub quality gate, structured test reports, CloudWatch and S3 evidence | Historical reliability/coverage analytics |
| Deployment lifecycle tests | Dedicated upgrade/idempotency/destroy assertions excluded | Evaluate explicit lifecycle test coverage |

## 4. Alternatives Considered and Decision Rationale

Only architecture-shaping alternatives are recorded here. Exact implementation choices should be documented in the LLD or in future ADRs.

| Decision area | Alternatives evaluated | Preferred approach | Rationale and trade-off |
|---|---|---|---|
| **Orchestration** | GitHub-only, new CodePipeline, mixed control plane | **GitHub as sole orchestration authority** | Keeps one PR gate and avoids duplicated workflow state. Requires reliable AWS execution-status correlation |
| **AWS test execution** | GitHub execution, central CodeBuild, existing customer pipeline and candidate-runner execution | **Open for architecture discussion** | Central CodeBuild is one option. Runner identity, safe test invocation, reporting and avoiding E2E overlap need validation |
| **Test execution topology** | Per-test projects, central executor, existing candidate runners | **Not yet selected** | Keep the mechanism simple and ensure runner-specific checks use the actual runner context |
| **Test selection** | Always run all, change-only, mandatory + rules, graph-based TIA | **Mandatory + explicit change rules** | Balances coverage, auditability and cost without a dependency-graph platform |
| **Selection ownership** | Trigger logic inside each test, separate service, Catalog registry/policy | **Catalog-owned policy and test metadata** | Enables test authors to declare applicability while keeping selection auditable and centrally validated |
| **Candidate validation** | Test released tag, build untracked temporary copy, controlled PR revision override | **Deploy exact PR revision in `org-test`** | Validates what is proposed for release; requires immutable identity and environment-restricted override |
| **Lifecycle** | Fully persistent, fully ephemeral, hybrid | **Persistent foundation + temporary candidate** | Avoids repeated account/network setup and reduces stale candidate contamination. Shared prerequisites need careful classification |
| **Concurrency** | Parallel shared environment, per-PR isolated environment, serialized AWS validation | **Serialize against shared Test environment** | Simplest safe Phase 1 model given one active candidate and teardown requirement; reduces throughput |
| **Evidence** | Console-only, GitHub-only, short-lived logs, durable archive | **Structured GitHub/AWS evidence plus S3 archive** | Provides debugging and release auditability beyond native report retention; incurs storage/operational cost |

**CodePipeline boundary:** GitHub remains the main orchestrator. The Golden Path uses the existing customer CodePipeline. No new test orchestration pipeline is planned; whether the existing customer pipeline can also support focused integration checks remains open.

## 5. Security, Cost and Operations

### 5.1. Security and trust boundaries

- **GitHub-to-AWS authentication:** Prefer short-lived GitHub OIDC credentials restricted to an approved repository/workflow context and named Test Toolchain resources. Do not expose unrestricted AWS access to arbitrary PR jobs.
- **Trusted execution control:** The chosen test launcher, buildspec where used, service roles, artifact policy, and approved targets are controlled independently of unreviewed candidate code. Prevent unauthorized role, source, and buildspec overrides.
- **Cross-account access:** Toolchain execution identities use least-privilege, auditable roles. Only explicitly approved Test account IDs may be targeted; no path to real production accounts is permitted.
- **Identity-fidelity tests:** Expected-denial and runner-boundary tests execute using the actual candidate runner principal and network context, not a privileged substitute.
- **Candidate artifacts:** Pin code and test bundles to immutable identities, validate integrity, encrypt at rest and in transit, and restrict read/write scopes.
- **Environment isolation:** Serialize shared AWS validation; keep state, artifacts, temporary resources and cleanup operations correlated to one execution.
- **Logs:** Redact credentials, secrets, tokens and sensitive outputs. Evidence access follows the organization's least-privilege policies.

### 5.2. Cost model and controls

Phase 1 cost is **TBD** and must be measured in the pilot. Primary drivers are Workshop Plan/Apply/Destroy runtime, candidate resource lifetime, AWS test execution, runner execution, CloudWatch ingestion, S3 evidence storage and data transfer. Cost controls include persistent shared foundations, mandatory candidate teardown, conservative test selection, build/test timeouts, one active candidate, and lifecycle policies for evidence.

Pilot metrics: cost per validated PR, total duration, AWS compute minutes, candidate resource-hours, test selection volume, failed-cleanup frequency, and evidence storage volume. The foundation-versus-candidate boundary is a budget-sensitive LLD decision.

### 5.3. Operational ownership

| Responsibility | Owner |
|---|---|
| Workflow orchestration, test selection, PR quality gate | Catalog / DevEx |
| Test definitions, assertions, applicability metadata | Test authors / Toolchain team |
| Plan, Apply, Destroy and environment reset | Workshop / Platform owners |
| Integration execution and test-result collection | Toolchain Test platform owner |
| Account baseline, network and cross-account security | Platform / relevant infrastructure owners |
| Release provenance and Runtime consumption enforcement | Catalog / Release / Runtime owners |

The team must be able to identify the exact failing test, execution mode, target account, candidate revision and diagnostic link without reconstructing the run manually. Failed cleanup requires quarantine, alerting, and a documented manual recovery owner.

## 6. Risks and Mitigations

| Risk | Consequence | Required control / pilot proof |
|---|---|---|
| Incomplete rule mapping | Relevant tests omitted | Mandatory baseline, policy validation, shared-path broad rules, conservative fallback |
| PR code changes trusted test execution behavior | Privilege escalation or false results | Separate trusted controls from unreviewed PR code; restrict roles, source/buildspec overrides and target permissions |
| Central privileged executor tests a runner denial | False security confidence | Run negative test with actual candidate runner identity; verify expected denial |
| Wrong candidate deployed or tested | Invalid release approval | Pin revision, verify deployed/tested/released provenance |
| AWS result not propagated | Failed build appears green in GitHub | Wait on exact execution IDs, require structured per-test evidence, fail closed |
| Shared environment collision or incomplete teardown | Flaky tests, cost leakage, cross-run contamination | Serialized AWS validation, lock through cleanup, quarantine on failed reset |
| Runner-specific integration tests cannot be executed safely | Missing critical coverage | Evaluate a controlled candidate-runner path or existing pipeline path before choosing the mechanism |
| Cleanup removes persistent foundation | Test environment outage | Explicit foundation/candidate ownership and state separation, pilot destroy rehearsal |
| Release content differs from tested content | Unvalidated version consumed | Provenance validation and Runtime allowlist/attestation control |
| Retention policy is too expensive or insufficient | Excess cost or missing evidence | Pilot measurements, S3 lifecycle, Security/Operations approval |
| AWS test invocation permissions are not approved | Blocked execution | Confirm the allowed invocation path and required security controls before pilot |

## 7. Rollout and Validation

### 7.1. Pilot

Use one internal Workshop test project and the dedicated four-account Test organization. The pilot must demonstrate the **whole contract**, not merely a successful build: candidate Plan/Apply, mandatory and rule-selected tests, representative AWS integration checks using an agreed safe execution path, limited Golden Path E2E through the existing customer Pipeline, structured reporting, failed-test propagation, cleanup, and a release-blocking verdict. The pilot will also verify how automated E2E handles manual approvals.

### 7.2. Readiness and acceptance gates

| Gate | Evidence required |
|---|---|
| **Before pilot** | Four-account readiness, candidate revision delivery, agreed integration execution approach and approved invocation permissions, environment lock/reset ownership, test registry validation |
| **Functional acceptance** | Tests selected for documented reasons; correct candidate deployed; runner-specific checks use the real identity; failures block the PR; Golden Path follows the existing Pipeline |
| **Operational acceptance** | Diagnostic links resolve, evidence archive is complete, cleanup succeeds after success and failure, failed reset quarantines environment |
| **Security acceptance** | PR cannot modify privileged buildspec/roles or target real production; approved cross-account scopes demonstrated |
| **Release acceptance** | Tested revision matches approved release content; unapproved version cannot be consumed by Runtime |
| **Cost acceptance** | Measured runtime and cost fit approved budget/thresholds, or lifecycle/selection are revised before rollout |

Pause rollout on candidate identity mismatch, unauthorized AWS access, missing results, runner-context substitution, failure to block a release, or repeated cleanup contamination. An exception to a required gate must be explicit, approved, time-limited, and auditable.

### 7.3. Rollout waves

1. **Readiness:** Finalize approvals and baseline, establish the execution contract, prove candidate injection and safe cleanup.
2. **Internal pilot:** Run end-to-end against one Workshop project, including representative failure and recovery paths.
3. **Catalog enforcement:** Make checks mandatory and implement release provenance / Runtime consumption controls.
4. **Expansion:** Add tests and fixtures through the same registry/contract, measure cost and reliability.
5. **Future Phase 2:** Introduce dependency-aware selection or additional lifecycle coverage only if pilot data justifies it.

## 8. Open Questions, Decisions, ADRs and Approvals

### 8.1. Open implementation questions

Most architectural principles are already defined. The table distinguishes **open architecture decisions** from implementation details and approvals.

| ID | Question / implementation boundary | Current position | Next step / status |
|---|---|---|---|
| OQ-01 | Approved GitHub-to-AWS test invocation and permissions | Existing runner-role permissions offer a possible AWS path, but the test workflow's authorization is not yet approved | Security / implementation review |
| OQ-02 | Exact Workshop candidate SHA override and revision checks | Candidate-only override in `org-test` is decided | Defined; LLD / pilot validation |
| OQ-03 | How to execute Integration Tests, including checks requiring actual candidate runner IAM/VPC | Execution mechanism not selected; central CodeBuild is one possible option | **Open; architecture discussion** |
| OQ-04 | Persistent foundation, ephemeral resource inventory and Terraform state boundary | Hybrid environment lifecycle is decided | Defined; LLD |
| OQ-05 | Executor count and role separation | Depends on OQ-03; no separate architecture question | Tracked under OQ-03 / LLD |
| OQ-06 | Test bundle delivery and integrity | Immutable test identity is required; delivery depends on chosen execution mechanism | LLD |
| OQ-07 | Detailed mandatory tests and path/component mappings | Catalog registry and mandatory-plus-rules selection are decided | Test design / pilot |
| OQ-08 | Final evidence retention, S3 lifecycle and cost limits | 12-month S3 evidence and 90-day CloudWatch logs are proposed | Pending Security / Operations approval |
| OQ-09 | Cleanup recovery after partial Apply or failed Destroy | Cleanup, reset checks and quarantine are required | Defined; LLD / pilot |
| OQ-10 | Release content equivalence and approved Runtime consumption | Provenance and enforcement are required | Release LLD / enforcement readiness |
| OQ-11 | Golden Path pilot fixture and automated handling of manual approvals | Existing customer Pipeline flow is understood: Source, Build, then configured Plan/Approval/Deploy stages | Pilot design; approval handling remains open |

### 8.2. Architectural decisions

| ID | Decision | Status |
|---|---|---|
| D-01 | GitHub Actions is the end-to-end orchestrator and required PR gate | Confirmed |
| D-02 | Retain v3 testing layers and ordered fail-fast stage boundaries | Confirmed |
| D-03 | Phase 1 selection combines mandatory baseline and change-based rules | Confirmed |
| D-04 | Catalog owns tests, test metadata, and selection policy; test authors declare applicability | Confirmed |
| D-05 | Candidate Toolchain revision is deployed via Workshop Plan/Apply under `AWS/org-test` | Confirmed principle |
| D-06 | Four AWS Test accounts simulate Toolchain, Dev, Staging and Prod roles | Confirmed |
| D-07 | AWS Integration execution mechanism is not yet selected; central CodeBuild is one possible option | Open for architecture discussion |
| D-08 | Runner-specific behavioral checks must use the real candidate runner identity and network where relevant | Confirmed requirement; execution mechanism open |
| D-09 | Golden Path is a separate customer-like repository for selected E2E checks | Confirmed |
| D-10 | Persistent foundation plus temporary candidate teardown after every run | Confirmed principle; inventory pending |
| D-11 | One active AWS candidate run at a time, lock retained through cleanup | Confirmed Phase 1 approach |
| D-12 | GitHub/AWS structured evidence, cleanup verification, and fail-closed gate | Confirmed principle |
| D-13 | Release provenance and Runtime approved-version enforcement are required | Confirmed principle; implementation pending |
| D-14 | TIA and explicit deployment lifecycle test scenarios are deferred beyond Phase 1 | Confirmed |
| D-15 | 12-month durable evidence and 90-day CloudWatch retention are proposed baselines | Proposed, approval pending |

### 8.3. Proposed ADR consolidation

Future ADRs should record the meaningful alternatives and consequences, without creating one ADR per minor configuration detail:

| ADR | Scope |
|---|---|
| **ADR-001: Testing Strategy and Selection** | Layered validation, mandatory/rule-based selection, author-owned metadata, TIA deferral |
| **ADR-002: AWS Test Execution Architecture** | Integration execution alternatives, runner-context fidelity and security boundaries; decision pending |
| **ADR-003: Candidate Deployment and Environment Lifecycle** | Candidate override, Workshop org-test, four-account simulation, hybrid teardown and serialization |
| **ADR-004: Evidence and Release Integrity** | Results, retention, cleanup gates, provenance, approved-version consumption |

### 8.4. Architecture review approvals

| Review area | Required reviewers | Status |
|---|---|---|
| Test layers, selection and extension contract | Catalog / DevEx / Toolchain test owners | Pending |
| Candidate deployment and environment lifecycle | Workshop / Platform owners | Pending |
| Integration execution approach and required AWS permissions | Cloud Security / Platform | Pending |
| Evidence, retention, release provenance | Operations / Release / Security | Pending |
| Pilot scope, budget and rollout | DevEx / Platform leadership | Pending |

## References

### Design baseline

- **Terragrunt Toolchain Automated Testing HLD v3**, internal source document, used as the structural and architectural baseline for this revision.
- Project architecture discussions and confirmed decisions concerning Catalog, Workshop `AWS/org-test`, four-account Test topology, candidate runners, Golden Path, lifecycle, and release gates.

### Vendor documentation used to validate feasibility and design boundaries

- [AWS CodeBuild: StartBuild API](https://docs.aws.amazon.com/codebuild/latest/APIReference/API_StartBuild.html). Supports controlled build invocation and documents override risks.
- [AWS CodeBuild: condition keys and preventing unauthorized buildspec overrides](https://docs.aws.amazon.com/codebuild/latest/userguide/action-context-keys.html).
- [AWS CodeBuild: test reporting](https://docs.aws.amazon.com/codebuild/latest/userguide/test-reporting.html). Supported structured formats and 30-day native report expiry.
- [AWS CodeBuild: report group export to S3](https://docs.aws.amazon.com/codebuild/latest/userguide/report-group-export-settings.html).
- [AWS CodeBuild: buildspec reference](https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html).
- [GitHub Actions: OIDC authentication in AWS](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws).
- [GitHub Actions: workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax).
- [AWS STS: AssumeRole](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html).

---

**Revision note (v5.0):** Keeps the v4.1 architecture, test layers, selection, candidate deployment, evidence, lifecycle, and release gates. Clarifies the existing customer Pipeline used by Golden Path E2E, and records Integration Test execution as an open implementation decision rather than a selected CodeBuild solution. Open questions and decision statuses are aligned with that distinction.

**Document boundary:** This is an architecture proposal, not an implementation specification. An architectural requirement is not represented as an already implemented capability. All pending approvals and LLD decisions are explicitly identified above.
