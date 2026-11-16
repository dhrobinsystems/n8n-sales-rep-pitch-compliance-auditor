# n8n-sales-rep-pitch-compliance-auditor

**Discovery and demo call transcripts → redacted speaker turns → OpenAI evaluation against MEDDPICC and call-conduct checkpoints → deterministic scores → HubSpot scorecard notes, rolling QA properties on each rep's profile, and a weekly coaching task.**

I built this n8n workflow for sales leaders and enablement teams who want consistent call QA without listening to every recording. Each completed discovery or demo call is redacted, then assessed checkpoint by checkpoint. The model says *what happened*; the workflow, not the model, turns that into a weighted score, critical gaps and a verdict. Results land in HubSpot where managers already coach: a scorecard note per call, rolling averages and trend on the rep's profile, and one coaching task per rep per week.

> **Safe by default.** `SANDBOX_MODE=true` and `DRY_RUN=true`. Out of the box, the workflow only uses pinned fictional calls, model answers and HubSpot records. It makes no transcription, OpenAI or HubSpot calls until you turn both switches off.

Built by DH Robin, Revenue Infrastructure Architect · dhRobin Systems · MIT licensed.

---

## Table of Contents

1. [Why this exists](#1-why-this-exists)
2. [Architecture](#2-architecture)
3. [Repository layout](#3-repository-layout)
4. [Prerequisites](#4-prerequisites)
5. [Call-completed webhook & transcription API](#5-call-completed-webhook--transcription-api)
6. [Transcript normalizer & sanitizer](#6-transcript-normalizer--sanitizer)
7. [Methodology checkpoints](#7-methodology-checkpoints)
8. [OpenAI evaluation prompt](#8-openai-evaluation-prompt)
9. [Scoring & verdicts](#9-scoring--verdicts)
10. [HubSpot rep profiles, notes & coaching tasks](#10-hubspot-rep-profiles-notes--coaching-tasks)
11. [Environment variables](#11-environment-variables)
12. [Node configuration guide](#12-node-configuration-guide)
13. [Deployment steps & testing stages](#13-deployment-steps--testing-stages)
14. [Security & privacy model](#14-security--privacy-model)
15. [Troubleshooting](#15-troubleshooting)
16. [Known limitations](#16-known-limitations)
17. [Disclaimer & license](#17-disclaimer--license)

---

## 1. Why this exists

- **Managers review a handful of calls a month.** The rest of the pipeline is coached on gut feel.
- **Methodology adoption is invisible.** Everyone was trained on MEDDPICC, but nobody can say which pillar each rep skips.
- **Compliance risks hide in recordings.** An unapproved discount or a promised release date only surfaces when the customer quotes it back.
- **LLM-only scoring isn't auditable.** A model that outputs "72/100" can't explain itself, and its numbers drift between runs.

My answer:
- Fixed checkpoints, weighted per call type.
- The model judges each checkpoint with evidence turns.
- Scores are computed in code.
- Results land in HubSpot where coaching already happens, with no transcript text and no customer PII.

## 2. Architecture

```mermaid
flowchart TD
    W[Call Completed Webhook] --> V[Verify & Unwrap Call Event<br/>HMAC-SHA256 · call-type filter]
    V --> AC{Accepted?}
    AC -- rejected / ignored --> Z[Audit Logger]
    AC -- yes --> IT{Inline Transcript?}
    IT -- yes --> PK[Package Call Transcript]
    IT -- no --> SF{Sandbox Fetch?}
    SF -- sandbox --> LF[Load Transcript Fixtures] --> PK
    SF -- live --> FG{Fetch via GET?}
    FG -- generic API --> TG[Fetch Transcript GET] --> PK
    FG -- Fireflies --> TP[Fetch Transcript POST] --> PK
    PK --> N[Transcript Normalizer & Sanitizer<br/>Rep / Colleague / Prospect · metrics · redaction · ledger]
    N --> IC{Is Audit Candidate?}
    IC -- skipped / failed --> Z
    IC -- yes --> BR[Build AI Request<br/>checkpoints for call type · strict schema]
    BR --> SA{Sandbox AI?}
    SA -- sandbox --> SX[Simulate MEDDPICC Evaluation] --> PS
    SA -- live --> OA[OpenAI Evaluate Call] --> PS[Parse & Score Evaluation<br/>validation · weighted scores · verdict]
    PS --> G[Group Audits by Rep]
    G --> IR{Is Rep Batch?}
    IR -- failures --> Z
    IR -- yes --> BL[Build HubSpot Lookup] --> SH{Sandbox HubSpot?}
    SH -- sandbox --> SL[Simulate HubSpot Lookup] --> PL
    SH -- live --> FO[Find Rep Owner] --> FP[Find Rep Profile] --> FT[Find Open Coaching Task] --> PL[Plan HubSpot Writes]
    PL --> HW{Has HubSpot Writes?}
    HW -- no --> RC
    HW -- yes --> LW{Live HubSpot Writes?}
    LW -- no --> SW[Simulate HubSpot Writes] --> RC
    LW -- yes --> WN{Write Notes?}
    WN -- yes --> CN[Create Scorecard Notes] --> WP
    WN -- no --> WP{Write Profile?}
    WP -- yes --> UP[Update Rep Profile] --> WT
    WP -- no --> WT{Write Task?}
    WT -- no --> RC
    WT -- yes --> CT{Create Task?}
    CT -- create --> CK[Create Coaching Task] --> RC
    CT -- update --> UK[Update Coaching Task] --> RC[Record HubSpot Writes<br/>audit ledger]
    RC --> Z
    ET[Error Trigger] --> EH[Sanitized Error Handler] --> EN{Notify?} --> EP[Post Execution Error Notice]
```

| Stage | Nodes |
|---|---|
| Trigger & transcript intake | `Call Completed Webhook`, `Verify & Unwrap Call Event`, `Fetch Transcript (GET/POST)`, `Package Call Transcript` |
| Normalize & redact | `Transcript Normalizer & Sanitizer` |
| OpenAI MEDDPICC evaluation | `Build AI Request`, `OpenAI Evaluate Call` (sandbox: `Simulate MEDDPICC Evaluation`) |
| Parse & score | `Parse & Score Evaluation`, `Group Audits by Rep` |
| HubSpot rep profile, notes & tasks | `Find Rep Owner`, `Find Rep Profile`, `Find Open Coaching Task`, `Plan HubSpot Writes`, `Create Scorecard Notes`, `Update Rep Profile`, `Create / Update Coaching Task`, `Record HubSpot Writes` (dry run: `Simulate HubSpot Writes`) |
| Audit & errors | `Audit Logger`, `Sanitized Error Handler` |

## 3. Repository layout

```
n8n-sales-rep-pitch-compliance-auditor/
├── README.md        ← this file
├── workflow.json    ← import into n8n (inactive, sandbox defaults, pinned fixtures)
├── DISCLAIMER.md
└── LICENSE          ← MIT
```

All logic lives in Code nodes inside `workflow.json`. There are no community nodes and no stored n8n credentials. Every secret comes from an environment variable.

## 4. Prerequisites

| Item | Notes |
|---|---|
| n8n | 1.x, self-hosted or cloud, with `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` and `NODE_FUNCTION_ALLOW_BUILTIN=crypto` |
| Transcription source | Any recorder or transcription service that can send a signed webhook (section 5). It must either include the transcript, or expose it via a REST endpoint or the Fireflies GraphQL API. |
| OpenAI | API key with access to a model that supports strict Structured Outputs (`json_schema`) |
| HubSpot | Private app token with owners read (`crm.objects.owners.read`), contacts read/write and deals read, plus the scopes HubSpot's Notes and Tasks API pages list for your portal |
| HubSpot rep profiles | One contact per rep, using the rep's work e-mail, with the custom properties in section 10 |
| Workflow settings | Concurrency of 1 for this workflow; Settings → Error Workflow = this workflow |

## 5. Call-completed webhook & transcription API

**Endpoint:** `POST https://YOUR_N8N_HOST/webhook/pitch-audit/call-completed`

**Signature:** `X-Timestamp: <unix seconds>` and `X-Signature: sha256=<hex HMAC-SHA256(TRANSCRIPTION_WEBHOOK_SECRET, X-Timestamp + "." + raw body)>`. It's compared in constant time. Outside the sandbox, timestamps older than `SIGNATURE_TOLERANCE_SECONDS` (300) are rejected. If your provider signs differently, put a small relay (or an n8n sub-workflow) in front that re-signs with this scheme. Don't weaken the check.

```json
{
  "event": "call.completed",
  "call_id": "QA-C1",
  "call_type": "discovery",
  "occurred_at": "2026-11-15T18:00:00Z",
  "provider": "generic",
  "rep": { "email": "sam.okafor@lumen-fixture.example", "name": "Sam Okafor" },
  "hubspot": { "deal_id": "fx-deal-201" },
  "transcript": {
    "title": "Northwind Fixture Traders - discovery",
    "started_at": "2026-11-10T15:00:00Z",
    "duration_seconds": 2700,
    "account_name": "Northwind Fixture Traders",
    "participants": [
      { "name": "Sam Okafor", "email": "sam.okafor@lumen-fixture.example", "role": "internal" },
      { "name": "Dana Whitfield", "email": "dana.whitfield@northwind-fixture.example", "role": "external" }
    ],
    "utterances": [
      { "speaker": "Sam Okafor", "text": "Thanks for joining…", "start_ms": 0 }
    ]
  }
}
```

**Fields:**
- **`event`:** only `call.completed` is processed.
- **`call_type`:** must be in `AUDIT_CALL_TYPES` (`discovery,demo`). If it's missing, it's inferred from the title ("discovery", "intro", "qualification" / "demo", "walkthrough"). Other types are ignored.
- **`rep.email`:** required. It identifies the audited rep in the transcript, in HubSpot owners and in rep profiles.
- **`hubspot.deal_id`:** optional. Notes and tasks are also attached to the deal.
- **`transcript`:** optional. Without it, the transcript is fetched by `call_id`:

| `provider` (or `TRANSCRIPTION_PROVIDER`) | Request | Expected response |
|---|---|---|
| `generic` | `GET {TRANSCRIPTION_API_BASE_URL}/calls/{call_id}/transcript` | the `transcript` object above (bare, or under `transcript`) |
| `fireflies` | `POST https://api.fireflies.ai/graphql` · `transcript(id)` with `sentences`, `speakers`, `meeting_attendees` | Fireflies GraphQL response |

Both use `Authorization: Bearer TRANSCRIPTION_API_KEY` (`TRANSCRIPTION_AUTH_MODE=basic` sends `Basic base64(TRANSCRIPTION_API_KEY)` for providers that expect `key:secret`). For other recorders, map their export to the generic shape, either in the provider's own webhook or in a relay.

## 6. Transcript normalizer & sanitizer

1. **Rep identification:** the speaker whose participant e-mail (then name) matches `rep.email` / `rep.name` becomes **Rep**.
   - Other internal participants (`role: internal`, or an e-mail in `INTERNAL_EMAIL_DOMAINS`) become **Colleague N**.
   - External participants become **Prospect N**, and unknown speakers **Speaker N**.
   - Calls where the rep can't be identified are skipped (`rep_not_identified`), because a score needs the right person.
2. **Skips** (logged as `call_skipped`):
   - `too_short`: under `MIN_CALL_DURATION_SECONDS` (600);
   - `too_few_turns`: under `MIN_TRANSCRIPT_TURNS` (8);
   - `no_prospect_speech`, `rep_did_not_speak`;
   - `duplicate_in_batch`, `already_audited`.
3. **Conversation metrics** (no AI): rep talk share (% of words), rep and prospect question counts, longest rep monologue (words), turns and duration. A talk share above `MAX_REP_TALK_SHARE` (60%) adds a note to the scorecard.
4. **Redaction** before OpenAI:
   - **Names:** participant names (and first/last names) become their labels.
   - **Companies:** prospect company names (account name, prospect e-mail domains) become `[PROSPECT_COMPANY]`, and your names (`OWN_COMPANY_NAMES`, internal domains) become `[OUR_COMPANY]`.
   - **Contact details and numbers:** e-mails, URLs, phone numbers, amounts (`$250,000`, `40k`, `a hundred and twenty`), percentages and long numbers become `[EMAIL]`, `[URL]`, `[PHONE]`, `[AMOUNT]`, `[PERCENT]` and `[NUMBER]`.
   - **Your own terms:** anything in `REDACT_TERMS` (person names there also cover their first and last names) becomes `[REDACTED]`.
   - **Competitors** listed in `COMPETITOR_NAMES` are kept.
5. **Length:** transcripts over `MAX_TRANSCRIPT_CHARS` (60,000) keep the first 60% and the last 40% of the budget, with an "omitted" marker, and the model is told.

## 7. Methodology checkpoints

The default rubric (`version 2026.11-meddpicc`) has 17 checkpoints. Weight 0 means the checkpoint isn't assessed for that call type. **R** = required, and a miss is a critical gap.

| Pillar | Checkpoint id | What good looks like | Discovery | Demo |
|---|---|---|---|---|
| Metrics | `metrics.quantified_impact` | Quantified the business impact or success metrics | 2 **R** | 1 |
| Metrics | `metrics.baseline` | Established the current baseline | 1 | 0.5 |
| Economic Buyer | `economic_buyer.identified` | Identified who owns budget and sign-off | 2 **R** | 1 |
| Economic Buyer | `economic_buyer.access` | Asked for or secured access to the economic buyer | 1 | 1.5 **R** |
| Decision Criteria | `decision_criteria.captured` | Captured technical, business and commercial criteria | 2 **R** | 1 |
| Decision Criteria | `decision_criteria.mapped` | Mapped what was shown to the stated criteria | 0 | 2 **R** |
| Decision Process | `decision_process.steps` | Clarified steps, people and timeline | 1.5 **R** | 1 |
| Decision Process | `decision_process.next_step` | Agreed a concrete, dated next step | 1 | 1.5 **R** |
| Paper Process | `paper_process.procurement` | Asked about procurement, legal and security review | 0.5 | 1.5 **R** |
| Identify Pain | `identify_pain.root_cause` | Uncovered the underlying pain and its cause | 2 **R** | 1 |
| Identify Pain | `identify_pain.cost_of_inaction` | Explored the cost of doing nothing | 1 | 0.5 |
| Champion | `champion.identified` | Identified a potential champion | 1 | 1 |
| Champion | `champion.tested` | Tested the champion (influence, access, personal win) | 0.5 | 1.5 |
| Competition | `competition.alternatives` | Asked which alternatives are considered, including doing nothing | 1.5 | 1 |
| Competition | `competition.differentiation` | Positioned differentiation against them | 0.5 | 1.5 |
| Call Conduct | `conduct.agenda_and_recording` | Set an agenda and confirmed recording | 1 **R** | 1 **R** |
| Call Conduct | `conduct.no_unapproved_commitments` | No unapproved discounts, pricing promises, roadmap dates or guarantees | 1.5 **R** | 1.5 **R** |

**Custom rubric:** set `METHODOLOGY_JSON` to a one-line JSON object with the same shape: `{"version":"…","pillars":{"id":"Label"},"checkpoints":[{"id","pillar","question","weights":{"discovery":1,"demo":1},"required":["discovery"]}]}`. You can add call types (for example `negotiation`) by adding weights and listing them in `AUDIT_CALL_TYPES`. An invalid override falls back to the default and flags `methodology_override_invalid`. Change `version` whenever you change the rubric, because it's printed on every scorecard.

## 8. OpenAI evaluation prompt

One **Chat Completions** request per call: `POST {OPENAI_API_BASE_URL}/chat/completions`, with `response_format.type = json_schema`, `strict: true` and `store: false`. The checkpoint ids are an `enum`, so the model can't invent checkpoints. The model is set by `OPENAI_MODEL` (default `gpt-4.1-mini`; use any current model with strict Structured Outputs). `OPENAI_TEMPERATURE` is only sent when set.

### System prompt

```text
You are a sales quality-assurance coach. You assess one REDACTED sales call transcript against methodology checkpoints (MEDDPICC plus call conduct). The audited seller is labelled "Rep", other sellers are "Colleague N", buyers are "Prospect N" (or "Speaker N" when the role is unknown).

Rules:
1. Assess every checkpoint in CHECKPOINTS exactly once, by id. Judge what the Rep did (a Colleague's help counts when the Rep led the call) and how the prospect responded.
2. status: "met" when the Rep clearly did it; "partial" when it was attempted but shallow, vague or left unconfirmed; "missed" when it wasn't done although the call allowed it; "not_applicable" only when the call genuinely gave no opportunity. Never use not_applicable for a checkpoint marked required.
3. Conduct checkpoints are compliance checks. For conduct.no_unapproved_commitments, "met" means the Rep made no unapproved discounts, pricing promises, roadmap dates or guarantees; "missed" means at least one was made.
4. evidence_turns: the [T#] numbers that support the status (empty when missed).
5. observation: one neutral sentence about what happened. coaching_tip: one specific, actionable sentence (empty when met). Paraphrase only: never quote more than six consecutive words, and never include names, company names, amounts, percentages or dates.
6. risk_flags: every unapproved discount, pricing promise, roadmap commitment, guarantee, competitor disparagement or misleading claim, with its turns and a neutral description. Empty when there are none.
7. strengths: up to three short phrases. coaching_actions: up to three, highest impact first, each tied to a checkpoint id.
8. call_summary: two neutral sentences about the call's purpose and outcome.
Assess behaviour on this call only; do not speculate about the Rep's skills or character.
```

### User message (sandbox demo call, redacted)

```text
CALL CONTEXT
type: demo · duration: 40 min · turns: 14 · speakers: Rep, Prospect 1, Prospect 2
metrics: rep talk share 72% · rep questions 3 · longest rep monologue 25 words

CHECKPOINTS (id — pillar — what good looks like — required?)
metrics.quantified_impact — Metrics — Quantified the business impact or the success metrics the prospect cares about — optional
metrics.baseline — Metrics — Established the current baseline (today's numbers, effort or cost) — optional
economic_buyer.identified — Economic Buyer — Identified who owns the budget and final sign-off — optional
economic_buyer.access — Economic Buyer — Asked for or secured access to the economic buyer — REQUIRED
decision_criteria.captured — Decision Criteria — Captured the prospect's decision criteria (technical, business, commercial) — optional
decision_criteria.mapped — Decision Criteria — Mapped what was shown to the prospect's stated criteria — REQUIRED
decision_process.steps — Decision Process — Clarified the decision steps, people involved and timeline — optional
decision_process.next_step — Decision Process — Agreed a concrete, dated next step — REQUIRED
paper_process.procurement — Paper Process — Asked about procurement, legal and security review steps — REQUIRED
identify_pain.root_cause — Identify Pain — Uncovered the underlying pain and what causes it — optional
identify_pain.cost_of_inaction — Identify Pain — Explored the cost of doing nothing — optional
champion.identified — Champion — Identified a potential champion — optional
champion.tested — Champion — Tested the champion (influence, access, personal win) — optional
competition.alternatives — Competition — Asked which alternatives are considered, including doing nothing — optional
competition.differentiation — Competition — Positioned differentiation against those alternatives — optional
conduct.agenda_and_recording — Call Conduct — Set an agenda and confirmed that the call is recorded — REQUIRED
conduct.no_unapproved_commitments — Call Conduct — Made no unapproved commitments (discounts, pricing promises, roadmap dates, guarantees) — REQUIRED

TRANSCRIPT (redacted)
[T1] Rep: Hi Prospect 1, Prospect 2. As agreed, this call is recorded. Agenda: I'll show the three workflows you ranked highest last time, then we'll agree next steps.
[T2] Prospect 1: Great.
[T3] Rep: You said audit trails were your first criterion. Here's how every change is logged and exported.
[T4] Prospect 2: That covers our compliance requirement.
[T5] Rep: Second, role-based access for the regional teams, which you listed as criterion two.
[T6] Prospect 1: Good, that's what we need.
[T7] Rep: Prospect 1, if this goes ahead, what does it change for you personally, and can you bring it to your steering committee?
[T8] Prospect 1: It removes a weekly manual report for my team, and yes, I present to the committee on the third.
[T9] Rep: Compared with what you use today, the difference is that every change is versioned automatically.
[T10] Prospect 2: Rivalsoft does some of that, but not versioning.
[T11] Rep: Would it make sense to have your CFO join a short business review?
[T12] Prospect 1: Maybe later, she's very busy.
[T13] Rep: Understood. Shall we set a follow-up for next Thursday to go through pricing?
[T14] Prospect 1: Thursday works.
```

### Response schema (strict, demo call)

```json
{
  "type": "object",
  "additionalProperties": false,
  "required": [
    "checkpoints",
    "risk_flags",
    "strengths",
    "coaching_actions",
    "call_summary"
  ],
  "properties": {
    "checkpoints": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": [
          "id",
          "status",
          "evidence_turns",
          "observation",
          "coaching_tip"
        ],
        "properties": {
          "id": {
            "type": "string",
            "enum": [
              "metrics.quantified_impact",
              "metrics.baseline",
              "economic_buyer.identified",
              "economic_buyer.access",
              "decision_criteria.captured",
              "decision_criteria.mapped",
              "decision_process.steps",
              "decision_process.next_step",
              "paper_process.procurement",
              "identify_pain.root_cause",
              "identify_pain.cost_of_inaction",
              "champion.identified",
              "champion.tested",
              "competition.alternatives",
              "competition.differentiation",
              "conduct.agenda_and_recording",
              "conduct.no_unapproved_commitments"
            ]
          },
          "status": {
            "type": "string",
            "enum": [
              "met",
              "partial",
              "missed",
              "not_applicable"
            ]
          },
          "evidence_turns": {
            "type": "array",
            "items": {
              "type": "integer"
            }
          },
          "observation": {
            "type": "string"
          },
          "coaching_tip": {
            "type": "string"
          }
        }
      }
    },
    "risk_flags": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": [
          "type",
          "evidence_turns",
          "description"
        ],
        "properties": {
          "type": {
            "type": "string",
            "enum": [
              "unapproved_discount",
              "pricing_promise",
              "roadmap_commitment",
              "guarantee",
              "competitor_disparagement",
              "misleading_claim",
              "other"
            ]
          },
          "evidence_turns": {
            "type": "array",
            "items": {
              "type": "integer"
            }
          },
          "description": {
            "type": "string"
          }
        }
      }
    },
    "strengths": {
      "type": "array",
      "items": {
        "type": "string"
      }
    },
    "coaching_actions": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": [
          "checkpoint_id",
          "action"
        ],
        "properties": {
          "checkpoint_id": {
            "type": "string",
            "enum": [
              "metrics.quantified_impact",
              "metrics.baseline",
              "economic_buyer.identified",
              "economic_buyer.access",
              "decision_criteria.captured",
              "decision_criteria.mapped",
              "decision_process.steps",
              "decision_process.next_step",
              "paper_process.procurement",
              "identify_pain.root_cause",
              "identify_pain.cost_of_inaction",
              "champion.identified",
              "champion.tested",
              "competition.alternatives",
              "competition.differentiation",
              "conduct.agenda_and_recording",
              "conduct.no_unapproved_commitments"
            ]
          },
          "action": {
            "type": "string"
          }
        }
      }
    },
    "call_summary": {
      "type": "string"
    }
  }
}
```

## 9. Scoring & verdicts

**Parse & Score Evaluation** treats the model answer as untrusted input:

- **Call-level failures:** HTTP errors (401, 429, 400), refusals, truncation (`finish_reason: length`) and invalid JSON produce `evaluation_failed`. Nothing is written, and the call stays open in the ledger.
- **Missing checkpoint** → `missed` (flag `missing_in_output`).
- **`met` without valid evidence turns** → `partial` (`met_without_evidence`).
- **`not_applicable` on a required checkpoint** → `missed` (`required_marked_na`).
- **Any risk flag** forces `conduct.no_unapproved_commitments` to `missed`.
- **Every text field is re-redacted.** Any field sharing 8 consecutive words with the transcript is blanked (`verbatim_overlap`).

**Score:**
- Credit per checkpoint: met = 1, partial = 0.5, missed = 0.
- `score = Σ(weight × credit) ÷ Σ(weight)` over the checkpoints assessed for the call type (not-applicable ones excluded), on a 0–100 scale.
- Pillar scores use the same formula per pillar.
- **Grade:** A ≥ 90 · B ≥ 80 · C ≥ 70 · D ≥ 60 · F.

| Verdict | Rule | Coaching task priority |
|---|---|---|
| `review_required` | any risk flag (unapproved discount, pricing promise, roadmap commitment, guarantee, competitor disparagement, misleading claim) | HIGH |
| `fail` | score < `FAIL_SCORE` (50), or 2+ critical gaps | HIGH |
| `needs_coaching` | score < `PASS_SCORE` (75), or any critical gap | MEDIUM |
| `pass` | otherwise | LOW (task only if `CREATE_TASK_FOR_PASSING_CALLS=true`) |

## 10. HubSpot rep profiles, notes & coaching tasks

**Lookups:**
1. **Owner:** `GET /crm/v3/owners?email=<rep e-mail>`. The owner `id` is used for task assignment, and as the note owner.
2. **Rep profile:** `POST /crm/v3/objects/contacts/search` with `email EQ <rep e-mail>`. This is one internal contact per rep (create it once, for example with lifecycle stage "Other" and a "Sales rep" list). Duplicates are flagged `duplicate_rep_profiles`.
3. **Open coaching task:** `POST /crm/v3/objects/tasks/search` filtered on `hubspot_owner_id`, the exact subject `[Sales QA] Coaching plan · <ISO week>`, and `hs_task_status NEQ COMPLETED`.

**Writes** (one item per rep, so several calls in one run don't overwrite each other):

| Write | Endpoint | Details |
|---|---|---|
| Scorecard note per call | `POST /crm/v3/objects/notes/batch/create` | HTML with score, grade, verdict, pillar table, compliance flags, critical gaps, coaching actions, strengths, conversation metrics, checkpoint list, QA ref · associations note→contact **202** (rep profile), note→deal **214** (optional) · not auto-retried |
| Rep-profile QA properties | `PATCH /crm/v3/objects/contacts/{id}` | rolling values below |
| Weekly coaching task | `POST /crm/v3/objects/tasks` or `PATCH /crm/v3/objects/tasks/{id}` | create: subject, body, `NOT_STARTED`, `TODO`, priority, due in `COACHING_TASK_DUE_DAYS` (3) at 16:00 UTC, owner = rep (or `COACH_OWNER_ID`), associations task→contact **204**, task→deal **216** · update: new calls go on top of the body, and priority only goes up |

**Rep-profile contact properties** (group "Sales QA"; the prefix is configurable with `QA_PROPERTY_PREFIX`, default `meddpicc_`):

| Property | Type | Value |
|---|---|---|
| `meddpicc_calls_audited` | Number | total audited calls |
| `meddpicc_avg_score` | Number | mean score of the last 10 audits |
| `meddpicc_last_score` / `meddpicc_last_verdict` / `meddpicc_last_call_type` / `meddpicc_last_audit_date` | Number / Dropdown (`pass`, `needs_coaching`, `fail`, `review_required`) / Single-line text / Date | latest audit |
| `meddpicc_trend` | Dropdown | `improving` / `declining` (mean of the last 3 vs the 3 before, ±5 points) · `steady` · `not_enough_data` |
| `meddpicc_pass_rate` | Number | % of `pass` in the last 10 |
| `meddpicc_weakest_pillar` / `meddpicc_strongest_pillar` | Single-line text | lowest / highest pillar average over the last 10 |
| `meddpicc_history` | Multi-line text | compact JSON of the last `QA_HISTORY_SIZE` (20) audits: `{r: QA ref, d: date, t: type, s: score, v: verdict, p: pillar scores}` |

**Idempotency:**
- **Audit ledger:** `pitchAuditLedger` in static data, live mode only. It stores note/profile/task completion per call. Only missing parts are retried, and a complete call is skipped.
- **Profile history:** a call whose QA ref is already in `meddpicc_history` is never counted twice, even if static data is lost.

**Holds** (shown in the audit log):
- `rep_owner_not_found`: no task.
- `rep_profile_not_found`: the note goes to the deal only, or is held.
- `no_record_for_scorecard_note`, `already_in_profile_history`, `no_task_owner`, and lookup failures.

## 11. Environment variables

Copy this block into your n8n environment and replace every placeholder.

```bash
# ── Safety switches (both must be "false" before anything is written) ──
SANDBOX_MODE=true
DRY_RUN=true

# ── Transcription source ──
TRANSCRIPTION_WEBHOOK_SECRET=YOUR_TRANSCRIPTION_WEBHOOK_SECRET
# generic (REST, section 5) or fireflies
TRANSCRIPTION_PROVIDER=generic
TRANSCRIPTION_API_BASE_URL=https://transcription.YOUR_PROVIDER.example/v1
TRANSCRIPTION_API_KEY=YOUR_TRANSCRIPTION_API_KEY
# bearer, or basic (base64 of TRANSCRIPTION_API_KEY)
TRANSCRIPTION_AUTH_MODE=bearer
SIGNATURE_TOLERANCE_SECONDS=300
AUDIT_CALL_TYPES=discovery,demo

# ── OpenAI ──
OPENAI_API_KEY=YOUR_OPENAI_API_KEY
OPENAI_API_BASE_URL=https://api.openai.com/v1
OPENAI_MODEL=gpt-4.1-mini
OPENAI_MAX_COMPLETION_TOKENS=3000
# Leave empty for models that reject temperature
OPENAI_TEMPERATURE=

# ── HubSpot ──
HUBSPOT_ACCESS_TOKEN=YOUR_HUBSPOT_API_KEY
HUBSPOT_API_BASE_URL=https://api.hubapi.com
QA_PROPERTY_PREFIX=meddpicc_
QA_HISTORY_SIZE=20
# Optional: assign every coaching task to one coach / manager owner id
COACH_OWNER_ID=
COACHING_TASK_DUE_DAYS=3
CREATE_TASK_FOR_PASSING_CALLS=false

# ── Methodology & scoring ──
# One line of JSON (section 7); empty = built-in MEDDPICC + conduct rubric
METHODOLOGY_JSON=
PASS_SCORE=75
FAIL_SCORE=50
MAX_REP_TALK_SHARE=60

# ── Your company & redaction ──
INTERNAL_EMAIL_DOMAINS=YOUR_COMPANY_EMAIL_DOMAIN
OWN_COMPANY_NAMES=YOUR_COMPANY_NAME,YOUR_PRODUCT_NAME
COMPETITOR_NAMES=YOUR_COMPETITOR_1,YOUR_COMPETITOR_2
REDACT_TERMS=
REDACT_PERCENTAGES=true
CALL_REF_SALT=YOUR_RANDOM_SALT

# ── Call selection ──
MIN_CALL_DURATION_SECONDS=600
MIN_TRANSCRIPT_TURNS=8
MAX_TRANSCRIPT_CHARS=60000
AUDIT_LEDGER_DAYS=90

# ── Error notices (optional) ──
SLACK_WEBHOOK_URL=YOUR_SLACK_WEBHOOK_URL
SLACK_NOTIFY_ON_MONITOR_ERROR=true

# ── Audit ──
AUDIT_LOG_TO_STDOUT=true
AUDIT_RING_BUFFER_SIZE=500

# ── n8n platform ──
N8N_BLOCK_ENV_ACCESS_IN_NODE=false
NODE_FUNCTION_ALLOW_BUILTIN=crypto
```

Flags accept only `true/1/yes/on` or `false/0/no/off`. Anything else keeps the safe default. List values that still start with `YOUR_` are ignored. Keep comments on their own lines. **Set `CALL_REF_SALT` once and never change it**, because the QA refs in HubSpot depend on it.

## 12. Node configuration guide

| Node | Type | Key settings |
|---|---|---|
| Call Completed Webhook | Webhook v2 | POST `pitch-audit/call-completed` · Respond immediately · **Raw Body on** |
| Verify & Unwrap Call Event | Code v2, all items | HMAC, timestamp window, event and call-type filters, fetch request |
| Fetch Transcript (GET / POST) | HTTP Request v4.2 | auth from `TRANSCRIPTION_*` · full response · never error · 3 tries |
| Package Call Transcript | Code v2, each item | generic or Fireflies → one call shape |
| Transcript Normalizer & Sanitizer | Code v2, all items | section 6 |
| Build AI Request | Code v2, each item | section 8 |
| OpenAI Evaluate Call | HTTP Request v4.2 | 3 tries × 5 s (429/5xx) · batching 1 request per 1.2 s · 120 s timeout |
| Parse & Score Evaluation | Code v2, each item | section 9 |
| Group Audits by Rep | Code v2, all items | one item per rep |
| Find Rep Owner / Find Rep Profile / Find Open Coaching Task | HTTP Request v4.2 | Bearer `HUBSPOT_ACCESS_TOKEN` · full response · never error · 3 tries |
| Plan HubSpot Writes | Code v2, each item | section 10 |
| Create Scorecard Notes / Create Coaching Task | HTTP Request v4.2 | not auto-retried (no duplicates) |
| Update Rep Profile / Update Coaching Task | HTTP Request v4.2 | PATCH · 3 tries |
| Record HubSpot Writes | Code v2, each item | per-call audit ledger |
| Audit Logger | Code v2, each item | `pitchAuditLog` ring buffer (not in sandbox) |
| Error Trigger → Sanitized Error Handler | Error Trigger v1 + Code v2 | redacted summary, optional Slack notice |

## 13. Deployment steps & testing stages

### Deployment steps

1. In n8n, go to **Workflows → Import from File** and select `workflow.json`. It imports **inactive**.
2. Set the environment variables (section 11) and restart n8n.
3. In HubSpot:
   - create a private app with the scopes in section 4;
   - create the "Sales QA" contact properties (section 10);
   - create one rep-profile contact per rep, using their work e-mail.
4. Set **Settings → Error Workflow** to this workflow, and concurrency to 1.
5. Run Stage 1, then point your recorder's call-completed webhook at the production URL for Stages 2–4.

### Stage 1 · Sandbox (pinned fixtures, no network)

`SANDBOX_MODE=true`, `DRY_RUN=true`. Execute **Call Completed Webhook** with its 8 pinned deliveries. The seller is "Lumen", and reps Sam, Priya and Alex are fictional. Sam and Priya have rep profiles; Priya already has an open coaching task for the week.

| Delivery | Expected |
|---|---|
| QA-C1 · Sam · discovery (inline) | **98 / A / pass** · names, an amount, a phone number and the CFO's name (via `REDACT_TERMS`) redacted · scorecard note on the profile and deal · no task for a pass |
| QA-C2 · Alex · discovery (**fetched** from the transcription API) | **9 / F / review_required** · discount promise, roadmap guarantee and competitor disparagement flagged · 6 critical gaps · rep talk share 74% · no rep profile → note on the deal only · **new HIGH coaching task** |
| QA-C3 · Priya · demo (inline) | **83 / B / needs_coaching**: paper process (required on demos) missed, economic-buyer access partial · **existing coaching task updated** (new call on top, priority MEDIUM) |
| QA-C4 · Sam · demo (inline) | **62 / D / fail**: criteria not mapped + paper process missed · grouped with QA-C1 → **one** profile update (calls audited 6 → 8, trend improving) and one new task |
| QA-C1 again | skipped · `duplicate_in_batch` |
| tampered signature | rejected · `signature_mismatch` |
| `internal_sync` call | ignored · `call_type_not_audited` |
| 3-minute call | skipped · `too_short` |

Open **Simulate HubSpot Writes (Dry Run)** and the plan's `writes` to read the exact note HTML, profile properties and task bodies. All people, companies, domains (`*.example`), IDs, numbers, transcripts, model answers and secrets are fictional.

### Stage 2 · Shadow (real APIs, HubSpot writes simulated)

`SANDBOX_MODE=false`, `DRY_RUN=true`. Send real calls from a test recorder:
1. Check **Transcript Normalizer & Sanitizer** for leftover names, and extend `REDACT_TERMS` or `OWN_COMPANY_NAMES`.
2. Check **Parse & Score Evaluation** against your own judgement on 10–20 calls. Tune weights or `PASS_SCORE` with `METHODOLOGY_JSON` if needed.

OpenAI and the HubSpot lookups are called in this stage; nothing is written.

### Stage 3 · Live in a HubSpot test portal

`DRY_RUN=false` in a HubSpot developer test account or sandbox. Confirm:
- notes appear on the rep profile and deal;
- profile properties roll up;
- the weekly task is created once, then updated;
- a replayed webhook doesn't duplicate anything.

### Stage 4 · Production

Start with one team and agree with managers how scores are used (section 14) before rolling out.

## 14. Security & privacy model

- **No secrets in the workflow.** The transcription, OpenAI and HubSpot credentials come from `$env`. The JSON holds placeholders and one fictional sandbox signing secret, used only while `SANDBOX_MODE=true`.
- **Fails closed.** Every webhook needs a valid HMAC and a fresh timestamp.
- **Redact before AI.** OpenAI receives redacted transcripts only, with `store: false`. Check your OpenAI organisation's data controls, and your customers' and employees' consent for recording and analysis.
- **HubSpot gets assessments, not conversations:** scores, paraphrased observations and coaching actions, re-redacted and checked for copied transcript text. No customer names, amounts or transcript lines.
- **Employee data is personal data.** Scorecards describe identifiable employees. Restrict who can see rep-profile contacts and coaching tasks, tell reps how audits work, and don't use scores as the sole basis for performance or employment decisions. The prompt tells the model to assess behaviour on the call only, not skills or character.
- **Redacted logs.** Audit records hold hashed call and rep refs, call type, scores, verdicts, critical-gap ids, risk-flag types and write results. Keys matching `authorization|token|secret|password|api key|cookie|signature|webhook url` are replaced. OpenAI (`sk-…`) and HubSpot (`pat-…`) tokens, Bearer/Basic credentials, JWTs, private keys and Slack webhook URLs are scrubbed from every string.
- **Sanitized errors.** The error branch drops stack traces, request options and bodies.
- **Restrict n8n execution data.** Executions contain raw webhook payloads and transcripts. Limit access and set a short retention period.

## 15. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `signature_mismatch` | Different secret, or the body was re-serialised | Use the same secret; keep **Raw Body** on; sign `timestamp + "." + raw body` |
| `timestamp_outside_tolerance` | Clock drift or delayed retries | Sync clocks (NTP); raise `SIGNATURE_TOLERANCE_SECONDS` slightly |
| `call_type_not_audited` | Missing `call_type`, no keyword in the title | Send `call_type`, or add the type to `AUDIT_CALL_TYPES` and the rubric weights |
| `rep_not_identified` | Rep e-mail not in the participants or utterances | Send participant e-mails, or match `rep.name` exactly |
| `transcript_not_ready` / `transcript_not_found` | Transcript still processing | Send the webhook after transcription completes, or replay it later |
| `openai_bad_request` | Model lacks strict `json_schema`, or rejects `temperature` | Choose a supported model; leave `OPENAI_TEMPERATURE` empty |
| `rep_profile_not_found` | No contact with the rep's e-mail | Create the rep-profile contact |
| `rep_owner_not_found` | Rep isn't a HubSpot user, or uses a different e-mail | Match the HubSpot user e-mail, or set `COACH_OWNER_ID` |
| HubSpot 400 on profile update | Custom properties missing or prefixed differently | Create them (section 10) or set `QA_PROPERTY_PREFIX` |
| Duplicate weekly tasks | Task subject edited by hand, or concurrency > 1 | Keep the generated subject; concurrency 1 |
| Scores feel too strict or lenient | Rubric weights or thresholds | Tune with `METHODOLOGY_JSON`, `PASS_SCORE` and `FAIL_SCORE` in Stage 2 |

## 16. Known limitations

- **The model can be wrong.** Statuses are evidence-backed judgements, not facts. Managers should review the scorecard before coaching, especially for `review_required` and `fail`.
- **Redaction is rule-based.** It covers participants, known companies, `REDACT_TERMS` and common number formats, not every name mentioned in passing.
- **English-centric.** The metrics (question marks, word counts) and the prompt assume English transcripts.
- **Batch note results.** If a batch note create is only partly successful (HTTP 207), no note is marked done, and a retry can create a duplicate for the calls that succeeded.
- **Static data isn't transactional.** Keep concurrency at 1.

---

## 17. Disclaimer & license

### ⚠️ Disclaimer & Liability Limitation

Use at your own risk. The code, workflows, and scripts in this repository (`n8n-sales-rep-pitch-compliance-auditor`) are provided for educational and architectural demonstration purposes.

* **No Liability:** The author and dhRobin Systems accept no responsibility or liability for any direct, indirect, incidental, or consequential damages that result from using, misusing, or deploying these scripts.
* **No Warranty:** All software, n8n JSON blueprints, JavaScript modules and prompts are provided "AS IS", without warranty of any kind, express or implied. This includes, but is not limited to, warranties of merchantability, fitness for a particular purpose, and non-infringement.
* **Production Testing Required:** Always test thoroughly in an isolated staging environment, such as a HubSpot developer test account or sandbox, with `SANDBOX_MODE=true` and `DRY_RUN=true` enabled, before you connect live transcription, HubSpot or OpenAI credentials. Treat every score, verdict and coaching note as a draft that a manager must review before it's acted on, and never use it as the sole basis for performance or employment decisions.
* **Third-Party APIs & Compliance:** You alone are responsible for complying with the terms of service, rate limits, data privacy regulations (GDPR/CCPA), employment and works-council rules, and security policies of HubSpot, OpenAI, your transcription provider and any connected services. That includes telling call participants and employees how recordings and transcripts are used, confirming a lawful basis and any consent required for recording calls and sending transcripts to an AI provider, keeping customer personal data out of CRM notes, and reviewing every automatically created or updated record.

The full disclaimer is also available in [DISCLAIMER.md](https://github.com/dhrobinsystems/n8n-sales-rep-pitch-compliance-auditor/blob/main/DISCLAIMER.md).

### License

Released under the MIT License. Copyright (c) 2026 dhRobin Systems. See [LICENSE](https://github.com/dhrobinsystems/n8n-sales-rep-pitch-compliance-auditor/blob/main/LICENSE).

MIT © 2026 dhRobin Systems

---

Full text: DISCLAIMER.md · Licensed under the MIT License. License: MIT © 2026 dhRobin Systems · Provided as-is, see the disclaimer.
