---
name: adlc-spec-review
description: Review a requirement spec document on a Paperclip issue for ADLC governance compliance (FR-07). Use when woken to review a spec on a critical or high priority issue. Checks all six required sections and posts a structured approval or changes-requested comment with per-section feedback.
---

# ADLC Spec Review

Review a requirement spec document on a `critical` or `high` priority Paperclip issue for ADLC governance compliance (FR-07). Validates all six required sections and produces a structured approval or changes-requested outcome.

## When to Use

Invoke this skill when:
- You are woken to review a spec document on a `critical` or `high` priority issue
- A developer has posted a spec document and tagged Dev Lead for review
- You are performing FR-07 governance review within the ADLC process

## SLA

One working day from the moment the spec document is posted or the review is requested.

## Inputs

- **Issue ID** — the Paperclip issue identifier (e.g., `TEC-1249`). Use `$PAPERCLIP_TASK_ID` if not provided.

## Required Sections

Each section is evaluated for presence AND substance. A section that exists but contains only placeholder text (`_TBD_`, `TBD`, `…`) is considered incomplete.

| Section | Passing bar |
|---|---|
| **Description** | Clear statement of what the feature or fix does |
| **System Model** | Names at least one component/service/layer affected and describes data flow or structural impact |
| **Business Rules** | At least one numbered constraint, validation rule, or edge case |
| **Acceptance Criteria** | At least one testable, independently verifiable condition (checkbox preferred) |
| **Assumptions** | At least one stated assumption or an explicit acknowledgement that none exist |
| **Open Questions** | At least one question or an explicit acknowledgement that none remain |

## Workflow

### Step 1 — Fetch the spec document

```bash
curl -sS "$PAPERCLIP_API_URL/api/issues/{issueId}/documents/spec" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY"
```

If the document does not exist (404), post a comment asking the assignee to publish a spec document first, and set the issue back to `in_progress` with assignee unchanged.

### Step 2 — Evaluate each section

For each of the six required sections:

1. Check whether the section heading is present in the document body.
2. Check whether the section content is non-empty and substantive (not just `_TBD_` or empty bullets).
3. Record the verdict: **PASS** or **FAIL** and (if FAIL) a one-line note on what is missing.

Keep your per-section evaluation notes — you will use them in Step 3.

### Step 3 — Produce the outcome

**If all six sections PASS:**

Post an approval comment and advance the issue:

```bash
COMMENT="$(cat <<'MD'
## Spec Approved

All six required sections are present and complete. Spec passes ADLC FR-07 review.

| Section | Status |
|---|---|
| Description | ✅ PASS |
| System Model | ✅ PASS |
| Business Rules | ✅ PASS |
| Acceptance Criteria | ✅ PASS |
| Assumptions | ✅ PASS |
| Open Questions | ✅ PASS |

Proceed to implementation subtask creation.
MD
)"

curl -sS -X PATCH "$PAPERCLIP_API_URL/api/issues/{issueId}" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -H "X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID" \
  -H "Content-Type: application/json" \
  --data "$(jq -n --arg comment "$COMMENT" '{status: "in_progress", comment: $comment}')"
```

Then set the issue to `in_review` or advance to the next ADLC stage as appropriate. If the issue is a spec-review task, mark it `done`.

**If any section FAILS:**

Post a changes-requested comment with targeted per-section feedback and move the issue back to `in_progress`:

```bash
COMMENT="$(cat <<'MD'
## Spec Changes Requested

The following sections require revision before this spec can be approved (ADLC FR-07):

| Section | Status | Feedback |
|---|---|---|
| Description | {✅ PASS / ❌ FAIL} | {feedback or —} |
| System Model | {✅ PASS / ❌ FAIL} | {feedback or —} |
| Business Rules | {✅ PASS / ❌ FAIL} | {feedback or —} |
| Acceptance Criteria | {✅ PASS / ❌ FAIL} | {feedback or —} |
| Assumptions | {✅ PASS / ❌ FAIL} | {feedback or —} |
| Open Questions | {✅ PASS / ❌ FAIL} | {feedback or —} |

Please update the [spec document]({prefix}/issues/{identifier}#document-spec) and re-request review when ready.
MD
)"

curl -sS -X PATCH "$PAPERCLIP_API_URL/api/issues/{issueId}" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -H "X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID" \
  -H "Content-Type: application/json" \
  --data "$(jq -n --arg comment "$COMMENT" '{status: "in_progress", comment: $comment}')"
```

Replace each `{feedback or —}` with the specific gap you identified (e.g. "No components named — describe which services are affected"). Do not write generic feedback. If a section passes, write `—` in the feedback column.

## Notes

- Always use `jq -n --arg comment "$COMMENT"` (or the `scripts/paperclip-issue-update.sh` helper) to build JSON. Never inline multiline markdown into a raw JSON string.
- The spec document key is always `spec`. Do not look for a different key.
- If the spec document was last updated after the review was requested, re-fetch it before evaluating — you want the latest revision.
- Evaluation must be deterministic and traceable: fill in the table for every outcome, even when approving.
- Do not skip sections or aggregate failures ("several sections are incomplete") — name each failing section explicitly.
- This skill is rigid: follow the checklist exactly. Do not adapt the six required sections based on issue type or perceived complexity.

---

*TEC Custom Skill — maintained by the Deltek Technical Services Engineering team.*
