---
name: scalepad-assessment-evaluator
description: Evaluate and populate ScalePad Lifecycle Manager client assessments using available ScalePad data. Use when asked to assess a client, answer assessment questions, fill a Technology Alignment Assessment or similar Lifecycle Manager assessment, add rationale/comments, compare against previous assessments, or audit evidence before updating ScalePad via MCP/API tools.
---

# ScalePad Assessment Evaluator

## Core Standard

Answer assessment questions only when there is defensible evidence. Do not treat a previous assessment answer as proof that nothing changed. Previous answers are context, not evidence by themselves.

For every answered question, leave an internal comment explaining:

- selected response
- evidence used
- caveats or weak spots
- whether the answer was carried forward from a prior assessment

If the available data does not support an answer, leave the question unanswered and state why.

## Workflow

### 1. Select the ScalePad environment

Use the ScalePad MCP/API server named by the user. If more than one ScalePad server is configured and the user does not specify one, choose the clearly established default or ask before writing.

When using an `execute-request` style tool, include the API key explicitly in the HAR request headers:

```json
{
  "headers": [
    { "name": "x-api-key", "value": "<api key>" }
  ]
}
```

For JSON writes, also include:

```json
{ "name": "Content-Type", "value": "application/json" }
```

Never print secrets in logs or final output.

### 2. Resolve the client

Find the client by name through Core API client search or the equivalent available tool.

Capture:

- client id
- exact display name
- lifecycle/customer status
- primary domain
- source/integration lineage
- asset counts if exposed

If a Lifecycle Manager endpoint rejects a client id that works in Core, do not assume the client is absent. List the relevant records and match by client label/name, then use the id embedded in the Lifecycle Manager result.

### 3. Find the target assessment

List Lifecycle Manager assessments and filter by client.

When the user asks for the newest assessment, sort by `record_created_at` unless they explicitly mean most recently edited, in which case sort by `updated_at`.

When the user names a template such as `Technology Alignment Assessment`, match by assessment `title` and confirm:

- assessment id
- title
- template id
- status
- record_created_at
- evaluated_at
- updated_at
- question_count
- question_answered_count
- current score

If there are multiple matching assessments, choose the newest matching assessment and mention any notable older matching assessment used for comparison.

### 4. Read the full assessment and prior assessment

Get the full target assessment. Also get the most relevant previous assessment with the same client and same assessment template/title when one exists.

Extract each question:

- category
- question id
- template question id
- title
- description
- scoring instructions
- criteria ids and labels
- current selected criterion
- `is_previously_selected` criterion
- public/internal comments

Use the target assessment's own `assessment_question_id` and `assessment_criterion_id` values for writes. Do not write old assessment ids into the new assessment.

### 5. Gather current evidence

Pull all relevant current data before answering. Prefer paginated API reads over samples; follow `next_cursor` until records are complete or the user-approved scope is reached.

Common data sources:

- Core client details
- Core hardware assets
- hardware lifecycle/warranty records
- SaaS assets
- service contracts
- Lifecycle Manager goals, initiatives, and action items
- meetings and deliverables when relevant
- ControlMap health metrics, assessment summary, risks, and action items
- Backup Radar clients/devices/jobs/backup health, when exposed
- service tickets, where exposed and relevant to help desk, ITSM, incidents, or monitoring

Summarize evidence into counts and facts that map to the assessment:

- asset counts by type
- operating systems and legacy/end-of-life systems
- antivirus status and definition status
- warranty expired/expiring/missing counts
- active service contracts and terms
- backup coverage and retention
- risk levels and severe/high risk counts
- compliance scores and control implementation state
- action items by status, priority, and initiative linkage
- roadmap, budget, vCIO, or governance artifacts

### 6. Decide each answer

For each question, classify the evidence:

- **Strong**: current ScalePad data directly supports the selected criterion.
- **Moderate**: current ScalePad data supports the general maturity level but not every criterion detail.
- **Weak**: only prior assessment, indirect evidence, or incomplete API data exists.
- **Unsupported**: no relevant evidence.

Rules:

- Answer Strong and Moderate questions.
- Answer Weak questions only if carrying forward is useful and the internal comment explicitly says it is weak or prior-assessment based.
- Leave Unsupported questions unanswered.
- Do not raise a maturity score unless current evidence supports the higher criterion.
- Do not preserve an old answer if current evidence contradicts it.
- If evidence is mixed, choose the lower defensible rating and explain the conflict.

Examples:

- Active network monitoring contract plus unresolved WIDS/NIDS action items usually supports `Needs Attention`, not `Satisfactory`.
- Microsoft 365 licensing alone does not prove mature cloud governance or cost optimization.
- A previous `Satisfactory` AI/ML answer is weak if no current AI/ML systems, usage, or measurable benefits are visible.
- Unknown AV telemetry, expired warranties, and legacy OS inventory are evidence against high endpoint-management maturity.

### 7. Write selected criteria

Use the assessment evaluation endpoint or equivalent tool to select criteria.

Payload shape commonly looks like:

```json
{
  "question_evaluations": [
    {
      "question_id": "<target assessment question id>",
      "selected_criteria_id": "<target assessment criterion id>"
    }
  ]
}
```

Write only the questions you intend to answer. Do not complete the assessment unless the user explicitly asks.

### 8. Add internal comments

Use internal comments by default. Public comments may be client-visible; use them only when the user asks.

Internal comment endpoint shape commonly looks like:

`PUT /lifecycle-manager/v1/assessments/{assessment_id}/questions/{question_id}/comment/internal`

Payload:

```json
{
  "comment_plain_text": "Selected Needs Attention. Evidence: ... Caveat: ..."
}
```

If rich text is required, include `comment_json` as valid ProseMirror JSON and ensure the plain text matches the extracted text.

Recommended comment format:

```text
Selected <rating>. Evidence: <current facts>. Caveat: <limits/contradictions>. Source: <Core/Lifecycle Manager/ControlMap/Backup Radar/prior assessment>.
```

For prior carry-forward:

```text
Selected <rating> from prior assessment. Current API review found <available data> but did not expose <missing evidence>; treat as prior-assessment carry-forward.
```

### 9. Verify after writing

Re-read the assessment and verify:

- selected-answer count
- score, if applicable
- per-category answered counts
- unanswered questions
- internal comment count on answered questions
- no answer was written without an internal comment

If any write failed, report exactly which question failed and whether selection or comment write failed.

## Evidence Mapping Guide

Use these mappings as prompts, not as automatic rules.

### Foundation Services

- Help Desk & End-User Support: service tickets, SLAs, support contracts, help desk tooling, self-service portal, CSAT, 24x7 coverage.
- Device & Endpoint Management: hardware inventory, RMM lineage, patching, MDM, encryption, warranty lifecycle, AV/EDR telemetry, legacy OS.
- Network Infrastructure: network assets, monitoring contracts, SNMP/topology/link saturation data, network risks/action items.
- Data Backup & Recovery: Backup Radar data, backup contracts, retention, restore testing, BCDR controls/action items.
- Security & Compliance: ControlMap health, risks, controls, evidence, compliance frameworks, IAM/EDR/MFA/DLP contracts or initiatives.
- Cloud Services: cloud/SaaS contracts, backup, governance, cost optimization, shared responsibility, cloud security findings.

### Business Applications

- Email & Collaboration: Microsoft 365/Google Workspace contracts, collaboration tools, DLP, retention, security configuration.
- BI & Analytics: BI tools, reporting automation, dashboards, data quality, analytics initiatives.
- CRM: CRM assets/contracts/integrations, data quality, sales/marketing workflow evidence.
- ERP: ERP assets/contracts/integrations, finance/operations process evidence.
- Industry Applications: line-of-business applications, compliance-specific tools, industry workflow evidence.

### Advanced Technologies

- AI/ML: actual AI tools, automation use cases, measurable benefits, governance.
- IoT: IoT assets/platforms, monitoring, security controls.
- RPA: automation tools, bot/process inventory, efficiency outcomes.
- Blockchain: actual blockchain/distributed-ledger implementation.

### Strategic Planning

- Technology Strategy & Roadmap: vCIO contracts, QBRs, roadmap artifacts, initiatives, goals, budgets.
- IT Governance & Risk Management: ControlMap governance, risk summaries, severe/high risks, policy/action-item state.
- Technology Investment & ROI: budget action items, refresh plans, ROI tracking, portfolio review.
- Digital Transformation Readiness: active initiatives, change management, blockers, adoption plans.

### Operational Excellence

- ITSM: ticket metrics, incidents, SLAs, action tracking, improvement process.
- Performance Monitoring & Optimization: monitoring contracts, alerting, telemetry, continuous monitoring, unresolved detection gaps.
- Capacity Planning & Scalability: capacity controls, failover, alternate processing, growth planning.
- QA & Testing: test processes, tabletop exercises, release QA, automated testing, defect management.

### Innovation & Future Readiness

- Innovation Culture & Processes: experiments, governance, innovation pipeline, funding, ownership.
- Emerging Technology Evaluation: pilots, evaluation process, roadmap entries for new technology.
- Technology Talent & Skills: training, certifications, role coverage, retention, staffing plans.
- Future Technology Vision: long-term roadmap, strategic advisory, target architecture, future-state initiatives.

## Final Response

Report:

- assessment id and title
- how many questions were answered
- how many remain unanswered
- whether internal comments were added and verified
- major evidence sources used
- any answers that are weak, prior-only, or need human review

Do not overstate certainty. Say exactly where the API did not expose enough evidence.
