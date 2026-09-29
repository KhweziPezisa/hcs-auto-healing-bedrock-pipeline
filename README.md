# HCS Auto-Healing Pipeline
 
An AI-assisted, human-approved auto-healing pipeline for Huawei Cloud Stack (HCS) infrastructure alarms, designed for a private cloud environment serving a banking client, on behalf of RedMPS.
 
**Repository:** TODO: REPO_URL not yet assigned.
**Mirror:** TODO: MIRROR_URL not yet assigned, if one is required.
**Maintainer:** Dumi, DevOps Engineer, RedMPS.
 
## Table of Contents
 
- [Overview](#overview)
- [Real-World Business Value](#real-world-business-value)
- [Skills Demonstrated](#skills-demonstrated)
- [Project Folder Structure](#project-folder-structure)
- [Pipeline Architecture and Data Flow](#pipeline-architecture-and-data-flow)
- [Tier Classification Logic](#tier-classification-logic)
- [Data Sensitivity and Masking Strategy](#data-sensitivity-and-masking-strategy)
- [Human-in-the-Loop Approval Workflow](#human-in-the-loop-approval-workflow)
- [Tasks and Implementation Steps](#tasks-and-implementation-steps)
- [Core Implementation Breakdown](#core-implementation-breakdown)
- [IAM Role and Permissions](#iam-role-and-permissions)
- [Project Features (Detailed Breakdown)](#project-features-detailed-breakdown)
- [Design Decisions and Highlights](#design-decisions-and-highlights)
- [Local Testing and Validation](#local-testing-and-validation)
- [Errors Encountered and Resolved](#errors-encountered-and-resolved)
- [Known Limitations and Next Steps](#known-limitations-and-next-steps)
- [Conclusion](#conclusion)
## Overview
 
This is the finalised design of an AI-driven auto-healing pipeline for a Huawei Cloud Stack (HCS) private cloud environment serving a banking client. In this engagement, RedMPS is the Supplier, the bank is the Client, and Huawei/xFusion act as the Operations and Maintenance (O&M) and Infrastructure providers.
 
The pipeline takes an HCS alarm, classifies it into one of three tiers of remediation confidence, masks sensitive fields before any data leaves the HCS platform, calls AWS Bedrock (Claude Sonnet 4.6) to perform root cause analysis and recommend a remediation action, and routes the recommendation to a human approver over Microsoft Teams before any action is taken. No alarm, at any tier, is auto-executed without human sign-off.
 
This repository is a working proof-of-concept build of that design, used to validate the classification logic, the masking strategy, and the end-to-end flow ahead of a client demonstration. It covers three representative alarm types, fed as manually triggered JSON payloads rather than a live poll of the ManageOne console, so that the proof of concept is reliable and repeatable rather than dependent on live infrastructure conditions on the day. The design itself — the tier logic, the masking rules, and the mandatory approval gate — is the finalised piece; the alarm coverage, the live ManageOne integration, and the feedback loop are the parts still to be built out. This README will be updated whenever the pipeline design or its implementation changes.
 
## Real-World Business Value
 
- Reduces the manual triage burden on the operations team by providing a structured root cause analysis and a recommended remediation action for every alarm, rather than requiring an engineer to start from the raw alarm and the Huawei help guide each time.
- Preserves human accountability for every remediation decision. The client architect's mandate is enforced in code: `requires_approval()` unconditionally returns `True` for all three tiers, so no alarm is ever auto-executed.
- Aligns approval urgency with business impact rather than technical classification. The approval service-level agreement (SLA) is driven solely by incident priority (P1 to P4), not by which tier the alarm was classified into.
- Keeps customer and internal infrastructure identifiers out of a third-party AI service by default. Sensitive fields are hashed before being sent to AWS Bedrock, and only released in the clear when a specific alarm type has been explicitly flagged as needing the real identifier to produce an accurate diagnosis.
- Produces a durable, append-only audit trail (`logs/audit_log.jsonl`) of every classification, recommendation, and human decision, supporting the compliance and auditability requirements of a banking environment.
- Provides a foundation for a feedback loop in which approver corrections are captured and, once a correction pattern recurs, promoted into deterministic classification rules rather than opaque model fine-tuning, keeping the system auditable and human-reviewable.
## Skills Demonstrated
 
- Designing a tiered decision-classification system with explicit, testable business rules rather than an opaque model-only classifier.
- Integrating a large language model (AWS Bedrock, Claude Sonnet 4.6) into an operational pipeline with tier-adapted prompting and a graceful degradation path when the model is unreachable.
- Implementing a data sensitivity and masking layer (HMAC-SHA256 field hashing) with conditional overrides driven by business logic, rather than a single blanket masking rule.
- Building a human-in-the-loop approval workflow using Microsoft Teams incoming webhooks, Adaptive Card-style MessageCards, and a lightweight local HTTP callback listener.
- Writing unit and regression tests for classification logic, including synthetic edge cases and a volume-weighted simulation against real alarm-frequency data.
- Iteratively verifying assumptions against primary source material (Huawei ManageOne help guide procedures) rather than relying on guesses, and correcting the design when that verification contradicted the initial assumption.
- Structuring a Python project with a clear separation of concerns (`core/` for pure business logic, `pipeline/` for orchestration and external integrations, `payloads/` for test fixtures) and package-relative imports.
- Practising secrets hygiene: environment variables loaded from a git-ignored `.env` file via `python-dotenv`, rather than embedding credentials in scripts.
## Project Folder Structure
 
```
demo/
├── .env.example
├── .gitignore
├── README.md
├── setup.sh
├── core/
│   ├── __init__.py
│   ├── classify.py
│   ├── scrubber.py
│   └── alarm_table.py
├── payloads/
│   ├── tier1_alarm.json
│   ├── tier2_alarm.json
│   └── tier3_alarm.json
├── pipeline/
│   ├── __init__.py
│   ├── bedrock_rca.py
│   ├── teams_notifier.py
│   ├── pipeline.py
│   └── verify_bedrock.py
└── logs/
    └── audit_log.jsonl        (created automatically at runtime, git-ignored)
```
 
`core/` contains pure business logic with no external network calls: tier classification, field masking, and the alarm table. `pipeline/` contains orchestration and everything that talks to an external system: AWS Bedrock, Microsoft Teams, and the command-line entry point. `payloads/` contains the three manually triggered alarm fixtures used to exercise the pipeline, one per tier, standing in for a live ManageOne feed until that integration is built. `logs/` holds the append-only audit log written at runtime and is excluded from version control.
 
## Pipeline Architecture and Data Flow
 
The pipeline executes six sequential steps for a single alarm, implemented in `pipeline/pipeline.py`:
 
1. **Load** the alarm JSON payload. In the finalised design this step is a live subscription or poll against ManageOne; the current implementation reads from `payloads/` as a stand-in.
2. **Classify** the alarm into TIER_1, TIER_2, or TIER_3 using `core/classify.py` and the alarm table in `core/alarm_table.py`.
3. **Scrub** sensitive fields from the raw alarm using `core/scrubber.py`, producing a sanitised copy safe to send to a third-party AI service.
4. **Run root cause analysis** by calling AWS Bedrock (`pipeline/bedrock_rca.py`), passing the scrubbed alarm, the relevant Huawei help guide text, and a tier-adapted prompt.
5. **Request approval** by posting a Microsoft Teams MessageCard (`pipeline/teams_notifier.py`) containing the recommendation, and waiting for a human decision either via a Teams button callback or a keyboard fallback, bounded by the priority-driven approval SLA.
6. **Process the decision** by executing the relevant playbook if approved, logging a rejection reason if rejected, or logging a timeout if the SLA is breached, and writing a full audit record to `logs/audit_log.jsonl`.
No step in the current build executes an action against real HCS infrastructure. Playbook execution is simulated (`SIMULATED_PLAYBOOKS` in `pipeline/pipeline.py`) so the flow can be exercised safely and repeatedly; wiring simulated playbooks up to real remediation actions (for example, via Ansible/SaltStack against HCS) is the remaining step to move this from proof of concept to production.
 
## Tier Classification Logic
 
Classification is implemented in `core/classify.py` and evaluated in a fixed rule order, first match wins. This logic is the finalised design and is not expected to change without a deliberate revision:
 
1. Alarm type not present in the alarm table → **TIER_3**.
2. `known_alarm` is `False` (an unreviewed draft alarm-table entry) → **TIER_3**, unconditionally.
3. `remediation_structure` is `AMBIGUOUS` → **TIER_3**, unconditionally, regardless of the alarm's configured `tier_default`.
4. `tier_default` is `TIER_1` → **TIER_1**.
5. `tier_default` is `TIER_2` → **TIER_2**.
6. Otherwise → **TIER_3**.
Each alarm type in the table carries a `remediation_structure` value, one of:
 
- `SINGLE` — one linear procedure with no branching.
- `SEQUENTIAL` — the procedure branches on command output, but every branch is deterministic and scriptable.
- `AMBIGUOUS` — the procedure branches on human interpretation of which underlying cause applies, and can never be safely automated to a single recommended path.
This three-value structure replaced an earlier boolean `multi_remediation_path` field, which could not distinguish a deterministic multi-step procedure (`SEQUENTIAL`) from a genuinely ambiguous one (`AMBIGUOUS`). The distinction mattered in practice: when the alarm table was reconciled against real Huawei ManageOne help guide procedures, two alarm types initially guessed as `AMBIGUOUS` were confirmed to be `SEQUENTIAL` once the actual handling procedure was reviewed, and correcting this materially changed the resulting tier distribution in simulation.
 
The `known_alarm` field exists as a defensive check: it guards against a draft or unreviewed alarm-table entry ever being auto-classified into TIER_1 or TIER_2 by mistake, for example through a manual table edit that set `tier_default` incorrectly before the entry had been reviewed.
 
Every tier, once assigned, still requires human approval before any action is taken. See [Human-in-the-Loop Approval Workflow](#human-in-the-loop-approval-workflow).
 
## Data Sensitivity and Masking Strategy
 
Implemented in `core/scrubber.py`. Every alarm field falls into one of four categories before being sent to AWS Bedrock:
 
| Category | Fields | Treatment |
|---|---|---|
| Safe | `alarm_type_id`, `alarm_serial_number`, `alarm_name`, `severity`, `priority`, `occurred`, `source_system_type`, `metrics` | Passed through unchanged. |
| Always confidential | `vdc_name`, `vdc_id` | Always hashed. Never eligible for override, under any circumstance, even when the alarm type is flagged `id_dependent_resolution`. |
| Conditionally sensitive | `instance_name`, `source_system`, `alarm_id` | Hashed by default. Passed through in the clear only when the specific alarm type is flagged `id_dependent_resolution = True`, because Bedrock needs the real identifier to produce an accurate diagnosis for that alarm type. |
| Stripped | `additional_info`, `location_info` | Never sent at all. These fields were judged to carry too much topology and customer-linked context to safely transmit, even hashed. |
 
Any field not explicitly categorised is hashed by default, on the principle that an unrecognised field should be treated conservatively rather than passed through unexamined.
 
Hashing uses HMAC-SHA256, keyed with a secret salt supplied via the `SCRUBBER_SALT` environment variable, truncated to twelve hexadecimal characters with a `hashed-` prefix. The current build falls back to a clearly labelled placeholder salt if `SCRUBBER_SALT` is not set, and prints a warning when it does so; this fallback is not suitable for production use.
 
This design directly reflects client instructions gathered during requirements: customer-linked service names remain confidential at all times, internal resource identifiers and instance names remain classified except where genuinely required for resolution accuracy, and no customer or operational data may leave the HCS platform in an unmasked form without a specific, alarm-type-level justification.
 
## Human-in-the-Loop Approval Workflow
 
Following explicit feedback from the client's AI solutions architect, the finalised design enforces a strict no-auto-execution policy: every alarm, at every tier, requires a human approval decision before any remediation action is taken. Tier 1 remains a category, distinguished by having the lightest and fastest-approving path, not by being exempt from approval.
 
`requires_approval()` in `core/classify.py` unconditionally returns `True` for all three tiers.
 
The approval SLA is driven solely by the alarm's incident priority (P1 to P4), not by its tier:
 
| Priority | Approval SLA |
|---|---|
| P1 | 15 minutes |
| P2 | 30 minutes |
| P3 | 120 minutes (2 hours) |
| P4 | No fixed SLA (Next Business Day) |
 
The approval channel is Microsoft Teams, chosen over Slack or ServiceNow for speed of setup and because the client's technical team already uses it. `pipeline/teams_notifier.py` posts a MessageCard to a Teams incoming webhook, themed by tier (green for Tier 1, amber for Tier 2, red for Tier 3), containing the alarm facts, the Bedrock root cause and recommended action, notes for the approver to verify, any stated risks, and Approve and Reject action buttons.
 
Clicking a button in Teams posts a callback to a local HTTP listener (`pipeline/pipeline.py`, `ApprovalHandler`, listening on port 8765) which records the decision. If the Teams webhook is not configured, the pipeline falls back to a manual keyboard prompt so the flow can still be exercised end to end.
 
If no decision is received within the priority-driven SLA window, the pipeline records a timeout, takes no action, and (in production) would notify the on-call engineer, consistent with the client instruction that the system should default to a safe action if approval times out.
 
## Tasks and Implementation Steps
 
1. Gather and confirm design requirements directly from the client across confidentiality, masking, approval SLA, timeout behaviour, model provider preference, cross-border data handling constraints, alarm coverage, and feedback-loop expectations.
2. Design and iterate the tier classification logic, including two rounds of logical corrections (see [Errors Encountered and Resolved](#errors-encountered-and-resolved)) and a defensive check against unreviewed alarm-table entries.
3. Reconcile the alarm table's `remediation_structure` values against real Huawei ManageOne help guide procedures rather than assumption, correcting misclassifications this exposed.
4. Incorporate the client architect's mandatory human-approval-gate requirement and flatten the approval SLA model to depend only on incident priority.
5. Design and implement the field-level data sensitivity and masking strategy in `core/scrubber.py`.
6. Implement the AWS Bedrock root cause analysis integration in `pipeline/bedrock_rca.py`, with tier-adapted prompting and a graceful fallback path.
7. Implement the Microsoft Teams approval notification and callback listener in `pipeline/teams_notifier.py` and `pipeline/pipeline.py`.
8. Build the end-to-end orchestrator (`pipeline/pipeline.py`) tying together classification, scrubbing, root cause analysis, approval, and audit logging.
9. Build three representative alarm payloads, one per tier, with accompanying manual-trigger instructions for simulating each condition on HCS, as a stand-in for the live ManageOne integration.
10. Write and run classification unit tests, including synthetic edge cases and a volume-weighted simulation against real alarm-frequency data.
11. Run a full local smoke test of the pipeline with Bedrock and Teams unavailable, confirming graceful fallback behaviour at every step.
12. Reorganise the project into the `core/` and `pipeline/` package structure documented here, with `python-dotenv` wired in for environment variable loading.
Remaining, to move from this proof of concept to the finalised production pipeline: live ManageOne alarm ingestion in place of manual payloads, the full alarm table covering every real alarm type, the approver-feedback learning loop, and wiring simulated playbooks to real remediation actions. See [Known Limitations and Next Steps](#known-limitations-and-next-steps).
 
## Core Implementation Breakdown
 
**`core/classify.py`** — Defines the `Tier`, `RemediationStructure`, and `IncidentPriority` enums, the `AlarmTableEntry` and `Alarm` dataclasses, the `classify()` function implementing the rule order described above, `requires_approval()`, and `get_approval_sla_minutes()`.
 
**`core/scrubber.py`** — Defines the four field-sensitivity sets described above, the `_hash_value()` HMAC-SHA256 helper, and the `scrub()` function that returns a sanitised copy of an alarm plus a `_scrubber_summary` describing what was hashed or redacted, for audit purposes.
 
**`core/alarm_table.py`** — Contains the current three-entry `ALARM_TABLE` and the corresponding `HELP_GUIDES` dictionary of Huawei-style handling procedure text, used as context for the Bedrock prompt. The full alarm table, built from the complete ManageOne alarm export and verified against real help guide procedures for every entry, is the outstanding item to reach production coverage.
 
**`pipeline/bedrock_rca.py`** — Builds a tier-adapted prompt (brief for Tier 1, standard for Tier 2, full ranked-cause analysis for Tier 3) and calls `boto3`'s `bedrock-runtime` client to invoke `anthropic.claude-sonnet-4-6`. Parses the model's JSON response, stripping markdown code fences if present, and falls back to a clearly labelled placeholder result if the call fails for any reason.
 
**`pipeline/teams_notifier.py`** — Builds a tier-themed Teams MessageCard with Approve and Reject action buttons, and posts it to the configured webhook. Falls back to printing the card to the console if no webhook is configured.
 
**`pipeline/pipeline.py`** — The orchestrator. Loads environment variables via `python-dotenv`, runs the six-step flow described above, starts and tears down the local approval-callback HTTP listener, executes the relevant playbook on approval, and appends a full audit record to `logs/audit_log.jsonl`.
 
**`pipeline/verify_bedrock.py`** — A standalone smoke test that sends a minimal prompt to Bedrock and reports success or a set of common troubleshooting steps on failure. Intended to be run once before any live session, before any alarm payload is fed through the pipeline.
 
## IAM Role and Permissions
 
The pipeline requires an AWS Identity and Access Management (IAM) principal (user or role) with, at minimum, permission to invoke the pipeline's Bedrock model:
 
```
TODO: A formal IAM policy JSON has not yet been drafted for this project.
At minimum the principal needs bedrock:InvokeModel scoped to the
anthropic.claude-sonnet-4-6 model ARN in the target region (af-south-1,
with us-east-1 as a fallback/testing region). Model access for
anthropic.claude-sonnet-4-6 must also be enabled in the AWS Bedrock
console for the account and region in use before InvokeModel calls
will succeed.
```
 
Credentials are supplied to the pipeline via environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`), loaded from a local `.env` file. No credentials are stored in source code, and `.env` is excluded from version control via `.gitignore`.
 
## Project Features (Detailed Breakdown)
 
- **Tiered classification with defensive checks** — TIER_1, TIER_2, and TIER_3 routing, with unconditional overrides for unknown alarm types, unreviewed alarm-table entries, and ambiguous remediation procedures, so that a data-entry mistake elsewhere in the system cannot silently cause an under-reviewed alarm to be treated as low-risk.
- **Tier-adapted AI prompting** — the depth and structure of the Bedrock prompt scales with tier, giving a brief single-action recommendation for Tier 1 and a full ranked, risk-annotated root cause analysis for Tier 3, so the approver receives a level of detail proportionate to the alarm's ambiguity.
- **Conditional field masking** — a four-category sensitivity model rather than a single blanket masking rule, allowing a specific alarm type to be flagged as needing a real resource identifier for accurate diagnosis, while never permitting customer-linked identifiers such as VDC name or VDC id to be released under any circumstance.
- **Mandatory, priority-driven approval gate** — no auto-execution at any tier; approval urgency is driven purely by business-defined incident priority, not by how confident the classification logic is in the recommended fix.
- **Graceful degradation** — if AWS Bedrock is unreachable, the pipeline does not fail; it returns a clearly labelled fallback recommendation directing the approver to investigate manually. If the Teams webhook is not configured, the pipeline falls back to a keyboard-driven approval prompt.
- **Append-only audit logging** — every pipeline run writes a structured JSON Lines record capturing the alarm, the tier, the model used, the recommendation, the confidence score, the human decision, and the scrubbing summary, supporting after-the-fact review and compliance evidence.
- **Reliable, reproducible alarm fixtures** — three manually triggered JSON payloads, one per tier, each carrying its own metadata describing how to trigger the equivalent real condition on HCS, standing in for the live ManageOne feed until that integration is built.
## Design Decisions and Highlights
 
**Manual alarm injection over live ManageOne polling, for now.** The current build feeds alarms as manually triggered JSON payload files rather than by polling the live ManageOne console. Live polling was assessed as introducing several independent failure points inappropriate for an early proof of concept: API authentication and token expiry, unpredictable alarm detection timing, the network and firewall path between the pipeline host and both ManageOne and Bedrock, and noise from unrelated real alarms firing during a session. Reliability was prioritised over the additional realism of a live feed; live ingestion remains the designed path once the pipeline moves towards production.
 
**Microsoft Teams over Slack or ServiceNow for the approval gate.** Teams was already available to the client's technical team and could be wired up quickly using incoming webhooks and simple HttpPOST action callbacks, without the additional setup overhead of a full ServiceNow integration at this stage.
 
**`RemediationStructure` enum over a boolean `multi_remediation_path` flag.** The original boolean could not distinguish a deterministic, scriptable multi-step procedure from a genuinely ambiguous one requiring human interpretation. This was not a hypothetical concern: verifying the alarm table against real Huawei help guide procedures reversed two initial classifications and confirmed the change materially affects the resulting tier distribution.
 
**`known_alarm` as an unconditional override, not a convention.** The alarm-table schema always intended unreviewed draft entries to default to `tier_default = TIER_3`, but the original `classify()` function never actually checked the `known_alarm` field, relying purely on that convention being followed elsewhere. This was identified as a latent risk and closed by making `known_alarm is False` an unconditional TIER_3 override, verified by dedicated regression tests.
 
**Flattened, priority-driven approval SLA.** An earlier design considered varying the approval SLA by tier as well as by priority, on the reasoning that a more deterministic fix could tolerate a shorter review window. Following explicit client architect feedback that no alarm should auto-execute, this was simplified so that approval urgency depends only on business-defined incident priority, avoiding an approval model that implicitly treated higher automation confidence as justification for lower human scrutiny. This is now the finalised approval model.
 
## Local Testing and Validation
 
Testing to date has focused on the classification logic and the end-to-end pipeline flow.
 
- **Synthetic edge-case tests** covering an unknown alarm type, clean Tier 1 and Tier 2 cases, a Tier 1 or Tier 2 default overridden to Tier 3 by an `AMBIGUOUS` remediation structure, a `known_alarm = False` override, and combinations of these conditions. All cases passed.
- **Approval gate regression tests** confirming `requires_approval()` returns `True` for all three tiers, that the SLA is identical across tiers for a given priority, that P4 has no fixed SLA, and that the SLA values trace back to the client-supplied SLA reference.
- **Volume-weighted simulation** against a reconstructed real alarm table of sixteen alarm types and their approximate weekly volumes, comparing the resulting Tier 1 / Tier 2 / Tier 3 distribution against the client's stated target split. This surfaced a genuine finding, not a code defect: the real-world Tier 3 (full root cause analysis) burden appears roughly double the proportion originally estimated in the architecture review, once alarm types were correctly classified against verified Huawei help guide procedures.
- **End-to-end pipeline smoke test**, run locally with the AWS Bedrock call and the Teams webhook both deliberately unavailable, confirming that all six pipeline steps execute in order, the fallback root cause analysis result is used, the keyboard approval fallback is triggered correctly, and a complete, accurate audit record is written.
```
TODO: No automated test suite (for example, pytest-based) currently
exists for this pipeline. Test scripts referenced above were run
manually during development. Formalising these as a committed test
suite, and adding automated coverage for core/scrubber.py directly,
is recommended before this moves beyond proof-of-concept scope.
```
 
## Errors Encountered and Resolved
 
**Tier-1-default alarm with an ambiguous multi-step procedure silently falling through to Tier 3.** Under the original literal classification logic, a Tier-1-default alarm with a multi-path remediation procedure fell through to TIER_3 rather than remaining at TIER_1 or moving to TIER_2, because only the TIER_1 branch checked for multiple paths. This behaviour was reviewed and confirmed as correct: a multi-path alarm should never be treated as a simple, deterministic fix regardless of its configured default tier.
 
**Tier-2-default alarms not checked for ambiguity at all.** The original logic's TIER_2 branch never checked the multi-path condition, meaning a genuinely ambiguous alarm configured with a TIER_2 default would incorrectly remain at TIER_2. Resolved by making the ambiguity check an unconditional override, evaluated before any tier-default check.
 
**`known_alarm` field present in the schema but never enforced in code.** The classification function relied on the convention that unreviewed draft alarm-table entries would always carry `tier_default = TIER_3`, without actually checking the `known_alarm` field. Resolved by adding `known_alarm is False` as an unconditional override to TIER_3, with dedicated regression tests to prevent this from regressing.
 
**Boolean remediation-path flag misclassifying real alarm types.** Six high-volume alarm types were initially flagged for manual verification against real Huawei help guide content, because a simple boolean could not express the difference between a deterministic multi-step procedure and a genuinely ambiguous one. On review of the actual handling procedures, two of the six initial guesses were reversed, and the classification scheme itself was changed from a boolean to the three-value `RemediationStructure` enum described above.
 
**AWS Bedrock model access page unavailable in the console.** During setup, the AWS console's dedicated Bedrock model access page was reported as retired or relocated. This was not resolved within this engagement; several alternative navigation paths were suggested (via the Providers section, via Foundation models, or via the console's global search), but the exact current console layout for the account in question was not confirmed.
 
## Known Limitations and Next Steps
 
- The alarm table (`core/alarm_table.py`) currently covers only three alarm types. The full alarm table, built from a genuine ManageOne export and verified against real help guide procedures for every entry, is the largest remaining item before this reaches production coverage.
- The real-data simulation indicates the true Tier 3 (full root cause analysis) burden may be roughly double the proportion originally estimated in the architecture review. The architecture review's headline claims around auto-execution rate and mean-time-to-resolution reduction should be revisited in light of both this finding and the mandatory human-approval-gate requirement, which removes auto-execution entirely.
- The accepted tier-classification flowchart has not yet been folded into the Logical Design Document, and the document's Tier 1 state machine and timeout-handling sections have not yet been revised to reflect the mandatory-approval-gate change.
- Alarm ingestion is manual in the current build. The finalised design calls for polling or subscribing to ManageOne directly, which introduces the reliability considerations described in [Design Decisions and Highlights](#design-decisions-and-highlights).
- The feedback loop by which approver decisions are captured and used to improve future recommendations (feedback-augmented retrieval, with a defined path to promoting a recurring correction into a deterministic rule) is designed but not yet implemented in this codebase.
- Playbook execution is currently simulated rather than wired to real remediation actions against HCS.
- No automated test suite is currently committed for the pipeline; see [Local Testing and Validation](#local-testing-and-validation).
- `TODO: PROJECT_NAME`, `TODO: REPO_URL`, and `TODO: MIRROR_URL` have not yet been finalised and should be filled in once the project is formally named and hosted.
- `TODO: HCS_VERSION` — the specific Huawei Cloud Stack version in use by the client has not been confirmed for this document.
## Conclusion
 
This document describes the finalised design of the auto-healing pipeline: alarm intake, risk-tiered classification with defensive safeguards against unreviewed or ambiguous cases, sensitive-field masking before any data reaches a third-party AI service, AI-assisted root cause analysis proportionate to alarm ambiguity, and a mandatory, priority-driven human approval gate before any action is taken. The codebase in this repository is a working proof of concept of that design, covering three alarm types end to end to validate the approach ahead of a client demonstration. The items listed under [Known Limitations and Next Steps](#known-limitations-and-next-steps) represent the concrete path from this proof of concept to the full production pipeline, and this README will be kept in step with the pipeline as that work proceeds.
