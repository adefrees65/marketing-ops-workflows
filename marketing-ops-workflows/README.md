# marketing-ops-workflows

Two production-shaped n8n workflows: a lead capture/qualification/routing engine, and an AI content approval engine. Both are reference implementations of architectures I designed and ran in production, rebuilt here in n8n so the decisions are readable.

The interesting part isn't the tooling. It's where the logic lives, what the AI is allowed to decide, and what happens when something goes wrong.

---

## What's here

```
workflows/
  lead-capture-qualify-route.json        7 nodes  · lead lifecycle decision layer
  brand-custodian-approval-engine.json  28 nodes · AI content approval, two flows
prompts/
  brand-custodian-system-prompt.md       the classifier's contract, versioned
examples/
  01-enterprise-demo-request.json        sample payloads for the lead flow
  ...
```

---

## 1. Lead capture, qualification and routing

One webhook in, one decision out. Four capture sources — inbound hand-raiser, content download, engagement threshold, event scan — feed a single workflow.

**Normalize → Score & Qualify → Is MQA? → handoff or nurture → Respond**

### Design decisions worth arguing about

**Four entry points, one workflow.** The four sources send different payload shapes. A `SOURCE_MAP` in the Normalize node is the only place that knows about those differences; everything downstream sees one canonical schema. The alternative — four near-identical workflows — is what this replaced in production, where the branches were 95% identical and quietly drifting apart. Adding a fifth source is a new entry in a map, not a new workflow.

**Almost no branching on the canvas.** There's exactly one visual branch: MQA or not. Segmentation, ownership, cadence and channel all resolve through a `ROUTING_TABLE` lookup rather than nested IF nodes. Canvas branches are where this kind of workflow rots — each one accumulates its own slightly different logic until nobody can state the rules. A table can be read in one screen.

**Order matters in segmentation.** An existing account owner outranks firmographics, always. Routing a named account to a new BDR is the fastest way to lose the sales team's trust in the whole system, and trust is the actual dependency — not the routing accuracy.

**Every decision writes an audit record.** Including the qualification reason. An MQA you can't explain is an MQA sales will argue with.

**One constant, one location.** The MQA threshold is declared once, next to the code that computes distance-from-threshold. Downstream nodes read the computed value instead of re-deriving it. An earlier version declared it twice; changing it upstream left the second copy silently reporting a stale number. No error, no crash — just quietly wrong data feeding reporting. That bug is why this file has the comment it has.

---

## 2. Brand Custodian — AI content approval engine

An LLM classifies submitted marketing assets against brand governance rules; the pipeline around it does everything else. Configured here for a creative/media production agency (video, photography, brand identity, event coverage).

**Two flows on one canvas, deliberately.**

*Intake (form trigger):* form → file staging → Claude classification → tiered Asana task → Airtable audit record → conditional Slack notification.

*Approval (webhook trigger):* Asana webhook → stage read → record lookup → file move + shared link → audit record update.

### Why two entry points and not one

These are two different events in one lifecycle, separated by human time. Someone submits an asset; a reviewer acts on it twenty minutes or five days later. A single execution can't span that — you'd hold a run open for days waiting on a person, which dies the moment the instance restarts.

So the intake flow ends at "task created, record logged," and state lives where it belongs: Asana owns the approval stage, Airtable owns the audit record. The webhook flow picks the thread back up by finding the record via the Asana task GID.

Note the contrast with the lead flow, which consolidates *four* entry points into one. Same principle, opposite conclusion: **consolidate on the event, not on the tool.** Four sources of one event merge. Two different events stay separate.

### What the AI decides, and what it doesn't

The model returns strict JSON: tier, brand_pass, confidence, asset_type, industry, vector, summary, flags, rationale. Everything after that is deterministic.

**Uncertainty escalates upward.** `auto < design < brand`. A false "needs review" costs minutes. A false "auto-approve" costs brand integrity. A submitter asserting their own asset is compliant does not lower the tier.

**Two decisions, never conflated.** `brand_pass` answers *is there a defect the submitter must fix.* `tier` answers *who needs to review this.* Needing sign-off, usage rights or legal clearance is a routing reason, not a defect. (See the build log below — this one cost me a rewrite.)

**Rejections are validated against a controlled vocabulary.** If the model sets `brand_pass: false`, it must cite at least one flag from a defined list of actual asset defects. If it can't, the code overrides the rejection, escalates the tier instead, and stamps `escalated_not_failed` so the override is visible. The model's *concern* is preserved; its *routing decision* is corrected.

**Some rules are code, not prompt.** Co-branded content can never auto-approve. That's enforced after the model returns, not requested of it. Governance rules I'm unwilling to leave to a model's discretion live where they can't drift.

**Parse failure fails safe.** Unparseable model output escalates to full human review, never to auto-approve.

### Shadow mode

The auto-approve tier ships disabled. The AI classifies and recommends from day one, but auto-approved assets still land at "Ready for Review" and post a confirmation request to the review channel. One config flag flips it live, once a human owner signs off on an accuracy bar.

The AI has to earn the right to auto-approve. That's a rollout decision, not a technical one, and it's the part I'd defend hardest.

### Inference over data entry

The intake form asks only what the submitter actually knows: region, funnel stage, whether it's partnership content. Asset type, client vertical, service line and the catalog summary are inferred by the model. Four fields moved from manual entry to inference — the form got shorter and the record got richer.

---

## Build log — what broke, and what it taught me

Kept because the failures are more instructive than the finished canvas.

**The spec assumed an API that doesn't exist.** The original design had the orchestration layer submitting an Asana intake form via API so the board's existing rules would fire. Asana's API can't submit forms. Tasks created via `POST /tasks` never trigger rules watching for "form submitted," so they landed on the board and sat there, unassigned. Not a bug in the build — a false assumption in the spec, found by building. *Integration specs describe what the UI can do, not what the API can do.*

**The classifier conflated two questions.** First working version bounced a co-branded client reel back to the submitter — and its own rationale gave it away: it said the asset needed usage-rights documentation and talent clearances. Those aren't defects, they're reasons a *reviewer* needs to see it. The effect was that the highest-stakes content was the least likely to reach a human. Fixed by splitting the outputs and adding the controlled-vocabulary guard above. *When a model's output drives routing, validate it against a schema you control — a plausible wrong answer is the expensive kind.*

**Approved By was reading the wrong person.** It looked up whoever triggered the webhook. The reviewer of record is the task's *assignee*. Switching to assignee removed an API call, removed a credential dependency, and produced a better default: no assignee means no human reviewed it, which is exactly when "AI (Auto)" is the truthful value.

**A lookup that finds nothing fails silently.** When the record search returns zero rows, downstream nodes simply don't run — no error, no alert, just an approval that never syncs. In production that branch needs to do something visible. Known gap, deliberately documented rather than quietly left.

**Box requires HTTPS OAuth redirects.** A localhost callback can't be registered, and the workaround — tunneling so the redirect is HTTPS — regenerates its URL on every restart, meaning re-registering the app each time. Not worth it for the least interesting node in the graph, so file storage is stubbed. Knowing why an integration is hard and choosing not to fight it is a decision, not an omission.

**Schema mismatches are silent until they aren't.** Airtable field names are case-sensitive, the API reports exactly one unknown field per request, and letting `typecast` invent select options on write leaves the schema undefined until data happens to arrive — which means validation can't help you and nobody can read the intended vocabulary off the table. Declaring the controlled vocabulary up front is the same instinct as the flag list in the classifier.

---

## Running it

```bash
npx n8n           # http://localhost:5678
```

Import a workflow via the canvas menu → **Import from File**.

The lead flow runs with **no credentials at all** — it's the decision layer and deliberately sends nothing:

```bash
curl -X POST http://localhost:5678/webhook-test/lead-capture \
  -H 'Content-Type: application/json' \
  -d @examples/01-enterprise-demo-request.json
```

The approval engine needs an Anthropic API key at minimum. Asana, Airtable and Slack are optional — stub them and the classification path still runs end to end.

Note `/webhook-test/` only listens while the editor is armed; `/webhook/` requires the workflow to be active.

---

## Scope and honesty

These are reference implementations, not products. The lead flow doesn't send anything — it returns the decision, because the decision layer is the part worth reviewing. The approval engine's file-storage and some notification steps are stubbed, and labeled as such on the canvas.

The classifier evaluates asset *metadata* — title, description, filename — not extracted file contents. A v2 would pull text out of the staged file and put it in the prompt.

The architectures ran in production. This is them rebuilt in n8n, in a personal sandbox, with no company data.

MIT.
