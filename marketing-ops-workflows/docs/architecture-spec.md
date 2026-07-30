# AI Content Operations — Workflow & Technical Flow

**Build specification for the implementing engineer** · v1.4 · June 2026

Stack: Slack · Asana · Airtable · Box · LLM API
All three review tiers create an Asana task. Existing automation is preserved, not replaced.

> **Naming conflict — resolve before build.** The existing Asana "Marketing Approvals" board uses "Tier One" (design reviewer) and "Tier Two" (brand reviewer). This system adds a third, auto-approve tier. A new Review Type value and one new workflow rule must be created so that every asset — including auto-approved ones — has a task record. Final naming convention to be confirmed.

*Sanitized for public release: company name, reviewer names, Slack channel names and vendor-specific IT items removed. Architecture, routing logic, schema and open items are unchanged from the internal version.*

---

## 1. System overview

```
User submits content
        ↓
Brand Custodian AI  ── validates brand · classifies tier · classifies asset type
        ↓
   fails brand check → returned + flagged to submitter
        ↓ passes
AI pre-fills and submits the Asana intake form (all three tiers, Review Type set by AI)
        ↓
Existing Asana workflows fire — routing · assignment · Slack alerts
        ↓ on task approval (human or AI)
File stored in Box → Box link returned to the orchestration layer
        ↓
Auto-log to Airtable — metadata · Box link · tier · status
        ↓
Done — asset in Box, indexed in Airtable, appears in the biweekly digest
```

### The three paths

**Auto-Approve — requires a new Review Type value.**
AI submits with Review Type = "Auto-Approve." A new Asana rule moves the task to Approved and sets Approval Stage = Approved immediately. No human review step.

*Shadow mode active:* a reviewer still confirms or flags each auto-approve recommendation, to train the model and measure accuracy. A feature flag prevents live auto-resolution until the brand reviewer signs off on an accuracy threshold.

**Design Review (Tier One).**
AI submits with Review Type = "Tier One." Existing rules fire: assign to the design reviewer, add submitter as collaborator, copy Asset Title to Project Name, set Approval Stage = "Ready for Review," DM the reviewer when due. Section-routing rules sync status as the reviewer updates the stage.

**Brand Review (Tier Two).**
AI submits with Review Type = "Tier Two." Existing rules fire: assign to the brand reviewer, add submitter as collaborator, copy Asset Title to Project Name, set Approval Stage = "Ready for Review," post to the design review channel. Existing due-date and overdue-escalation rules handle reminders.

### Why three tiers

Review effort is allocated in proportion to **exposure**, not effort. Asset type is a proxy; visibility is the criterion. Most output is low-exposure and template-driven and does not warrant a director's attention — and when it consumes that attention, genuinely high-stakes work receives less. Automating the low end concentrates human judgment where it changes the outcome rather than removing it from the process.

This is why uncertainty always escalates upward, and why partnership and co-branded content is excluded from auto-approve regardless of how routine the asset appears.

---

## 2. Asana — "Marketing Approvals" board

*Existing configuration. Do not modify without review.*

**Board sections:** To Be Reviewed · Needs Information · Changes Requested · Approved · Rejected · Archive

**Intake form fields:** Name of submitter (text) · Email Address (email) · Asset Title (text) · Date needed by (date) · Asset description (long text) · Review Type (single select: Tier One · Tier Two · **+ Auto-Approve, to add**)

**Other task fields:** Priority (Low/Medium/High) · Project Name (= Asset Title, auto-set by intake workflow) · Approval Stage (Ready for Review · Changes Needed · Approved · Rejected · Needs Information)

### Required Asana changes

**One new Review Type value:** "Auto-Approve." This is the only intake form edit needed to support the new tier.

**One new workflow rule (auto-approve path).** When Review Type = "Auto-Approve" on form submit: set Approval Stage = Approved · move task to the Approved section · add submitter as collaborator · copy Asset Title to Project Name.

*During shadow review, the task is created but Approval Stage is set to "Ready for Review" instead — a reviewer confirms or flags before the task auto-resolves.*

**Recommendation:** add "Asset Type" to the intake form so the AI can classify it once and pass the value straight through to Airtable, eliminating double-entry for the submitter. Requires form field approval.

---

## 3. Asana — full automated workflow inventory

Seven active rules. All are preserved by the AI integration; none are replaced.

| Rule | Trigger | Condition | Actions |
|---|---|---|---|
| Route Tier One | Form submitted | Review Type = "Tier One" | Assign to design reviewer · add submitter as collaborator · copy Asset Title to Project Name · set Approval Stage = "Ready for Review" · move to "To Be Reviewed" |
| Route Tier Two | Form submitted | Review Type = "Tier Two" | Assign to brand reviewer · add submitter as collaborator · copy Asset Title to Project Name · set Approval Stage = "Ready for Review" · post to review channel |
| Approval status routing | Approval Status changed by reviewer | Approved / Changes requested / Rejected | **Approved:** move to Approved + set stage Approved · **Changes requested:** move to Changes Requested + set stage Changes Needed · **Rejected:** set stage Rejected + move to Rejected |
| Needs Information routing | Stage changed or task moved | Stage = "Needs Information" OR Section = "Needs Information" | Move to Needs Information section · create subtask for submitter |
| Due date alert — design | Task due today | Assignee = design reviewer | Slack DM: "Approval ALERT: [task] needs approval by [due date]" |
| Due date alert — brand | Task due today | Assignee = brand reviewer | Channel message: "[task] is due today" |
| Overdue escalation | Task overdue by 1 day | Assignee = brand reviewer | Channel message: "[task] is now overdue for approval" |

**AI integration principle.** The orchestration layer submits the intake form via API with all fields pre-filled. This triggers every rule above without modification. The only net-new rule required is the auto-approve path.

**Webhook trigger.** On Approval Stage changing to "Approved," the orchestration layer fires: upload file to Box → retrieve Box link → write record to Airtable. This is the connection the implementing engineer needs to build.

---

## 4. Airtable — schema

Existing fields unchanged. New fields support the AI logging layer.

| Field | Type | Values / format | Source | Status |
|---|---|---|---|---|
| Name | Text | Asset name | Requestor input | EXISTING |
| Asset Link | URL | Box file URL | Box API — auto-populated on upload | EXISTING |
| Asset Type | Single select | One-pager · landing page · deck · etc. | Requestor input — *propose: AI auto-classify* | EXISTING |
| Summary | Long text | Brief context / asset coverage | Requestor input — *propose: AI-generated first draft* | EXISTING |
| Upload Date | Date | Date uploaded / created | Auto-timestamp | EXISTING |
| Region | Multi-select | Geographic region(s) | Requestor input | EXISTING |
| Industry | Multi-select | Industries targeted | Requestor input or AI inference | EXISTING |
| Vector | Multi-select | Solution / service axis | Requestor input or AI inference | EXISTING |
| Funnel Stage | Multi-select | TOFU · MOFU · BOFU | Requestor input or AI inference | EXISTING |
| Partnership Content | Checkbox | Yes / No | Requestor input | EXISTING |
| Created By | Person | Name of uploader | Auto from account | EXISTING |
| **Approval Tier** | Single select | Auto-Approve · Design Review · Brand Review | AI classification output | **PROPOSED** |
| **Approval Status** | Single select | Pending · Approved · Changes Requested · Rejected | Asana webhook — auto-updated on task change | **PROPOSED** |
| **AI Confidence Score** | Number | 0.0 – 1.0 | Brand Custodian AI output | **PROPOSED** |
| **AI Flags** | Long text | e.g. "tone," "logo usage" | Brand Custodian AI output | **PROPOSED** |
| **Asana Task Link** | URL | Link to Asana task (all tiers) | Asana API — on task creation | **PROPOSED** |
| **Approved By** | Text | Reviewer name, or "AI (Auto)" | Asana completion webhook | **PROPOSED** |

**Existing workflow preserved.** Airtable sends a biweekly digest listing all new assets since the last cycle. AI-logged records appear automatically in the next digest — no change required.

**IT open items.** SSO configuration and API write access must be confirmed before automation goes live. Search-restriction scope on this base needs clarification from IT before build starts.

---

## 5. Technical integration

### Brand Custodian AI

Validates content against the Messaging & Brand Governance SOP. Returns structured JSON to the orchestration layer:

```json
{ "tier": "auto" | "design" | "brand",
  "brand_pass": true,
  "confidence": 0.0,
  "asset_type": "one_pager",
  "flags": ["tone", "logo_usage"] }
```

### Asana API — workflow execution

The AI pre-fills and submits the intake form via API for all three tiers, triggering all existing downstream workflows without modification. A completion webhook fires on Approval Stage = "Approved," which triggers Box upload and Airtable logging. Existing board structure is not modified.

### Slack — notifications

Auto-Approve: silent, no notification. Design Review: reviewer DM, handled by the existing Asana due-date workflow. Brand Review: channel messages handled by existing Asana workflows (form submit, due date, overdue). **No new permanent Slack channels.** All existing governance preserved.

### Box — `files.upload`

Uploaded at intake. Box share URL written to the Airtable "Asset Link" field. The file is never stored in Airtable. Folder hierarchy to be defined — suggested: `/AI-Approved/{tier}/{YYYY-MM}/`

### Airtable — `records.create`

Existing fields plus the six proposed fields above. The biweekly digest is untouched. IT sign-off required on write access before build.

---

## 6. Open items before build

1. **Naming convention** — align existing Tier One / Tier Two labels with the proposed tier naming; add "Auto-Approve" as a Review Type before any routing logic is coded.
2. **New Asana workflow (auto-approve path)** — one new rule in the Asana rules editor.
3. **Airtable IT sign-off** — SSO configuration and API write access confirmed; search-restriction scope clarified.
4. **Brand Review content-type list** — the brand reviewer finalizes which asset types route to Brand Review. Required before routing logic locks.
5. **Asset type merge** — add Asset Type to the Asana form so the AI can classify once and pass the value to Airtable, eliminating double-entry.
6. **Orchestration middleware** — custom build, n8n, Make, or other. Recommendation needed based on the existing stack.
7. **Box folder structure** — define the folder hierarchy for AI-approved assets before the Box integration is built.
8. **Auto-approve accuracy threshold** — the brand reviewer defines the bar before the shadow-review feature flag flips to live auto-resolution.
9. **Brand Custodian maintenance owner** — who updates the AI knowledge base as brand guidelines change.

---

## 7. What building it changed

*Added after the fact. This specification stalled on open item #3 (IT sign-off). Rather than leave the design unvalidated, I implemented it independently in n8n — see `workflows/brand-custodian-approval-engine.json`. Building it surfaced four things this document got wrong or left underspecified.*

**§3 assumes an API that doesn't exist.** The integration principle — "the orchestration layer submits the intake form via API, triggering every existing rule without modification" — is not achievable. Asana's API cannot submit forms. Tasks created via `POST /tasks` never trigger rules whose trigger is "form submitted," so they land on the board unassigned and stall. The fix is to change those two rules' trigger to "task added to project" with the same conditions, which fires for both human and API-created tasks and preserves every action. **This would have been discovered after the integration was built, not before.**

**§5's JSON contract conflates two decisions.** `brand_pass` and `tier` are shown as independent fields but the contract never says what separates them, and in implementation the model collapsed them — rejecting a co-branded asset because it "required usage-rights documentation and talent clearances." Those are reasons a reviewer must see something, not defects the submitter can fix. The effect is that the highest-exposure content becomes the least likely to reach a human. The contract now defines `brand_pass` strictly as *is there a defect the submitter must fix*, and rejections must cite a flag from a controlled vocabulary of actual defects. A rejection citing none is overridden to a pass with the tier escalated instead.

**Some governance can't live in the prompt.** This spec states that partnership content is never auto-approved, but states it as a rule the model should follow. In implementation it is enforced after the model returns. Rules that must not drift belong in code, not in a prompt.

**Two entry points, not one.** This document reads as a single linear flow. In implementation it must be two: intake ends at "task created, record logged," and a webhook-driven flow resumes when a reviewer changes the stage. The two events are separated by human time, and no single execution should be held open across it. State lives in Asana and Airtable between them.

**Still open in the implementation:** a record lookup that returns nothing ends the run silently — no error, no alert, an approval that never syncs. In production that branch needs to raise an exception visibly.
