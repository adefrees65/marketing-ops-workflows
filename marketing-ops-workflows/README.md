# Lead Capture, Qualification and Routing

A reference implementation of a lead lifecycle in [n8n](https://n8n.io): one webhook in, a qualification decision out, with the routing rules written down instead of argued about.

The architecture here is one I designed and ran in production on a HubSpot / Slack / SalesLoft stack. This repo is that architecture rebuilt in n8n as a self-contained, runnable example — no credentials, no external calls, nothing to configure. Import it and it works.

## The problem it models

Most lead flows fail the same way. Marketing builds a separate workflow per capture source, each one accumulates its own slightly different follow-up logic, and within two quarters nobody can answer the only question that matters: *what makes a lead qualified?* Sales stops trusting the routing, marketing stops trusting the reporting, and the argument that follows is never really about workflows. It's about definitions nobody owns.

So this build starts from the definition and works outward.

```
Webhook  ->  Normalize Lead  ->  Score & Qualify  ->  Is MQA?  ->  Build Sales Handoff  -> Respond
                                                              \-> Assign Nurture Track -/
```

Seven nodes. Four capture sources. One set of rules.

## Architecture decisions

**One workflow, four entry points.** Inbound hand raisers, content downloads, engagement thresholds and events all hit the same webhook. `Normalize Lead` is the only node that knows their payloads differ — it maps each source's field names into one canonical schema. Adding a fifth source is a new entry in `SOURCE_MAP`, not a new workflow to keep in sync.

**The branching lives in code, not on the canvas.** There is exactly one visible branch, and it's the one that carries business meaning: qualified or not. Segment routing happens inside `Score & Qualify` as a lookup table. Canvas branches are where workflows rot — each one quietly grows its own version of the rules until the diagram no longer describes the system. A routing table is one screen you can read top to bottom, and adding a segment is a row.

**The MQA definition is a constant, not an opinion.** `MQA_MINUTES_THRESHOLD = 100` over a rolling three-month window, or any high-value conversion event. It sits at the top of the node, in caps, matching the SOP word for word. A threshold you can point at ends the argument. A threshold buried in a filter condition three clicks deep restarts it every quarter.

**Every decision records why it was made.** `qualification_reason` is written on every lead, qualified or not — `high_value_event:demo_request`, `engagement_threshold:118min`, `below_threshold:20min`. An MQA you can't explain is an MQA sales will argue with, and a funnel you can't reconstruct is a funnel you can't debug.

**Existing ownership outranks firmographics.** In `Score & Qualify`, the segment check tests for a named account owner *before* it looks at headcount. Routing someone's named account to a new BDR is the fastest way to lose the sales team's trust in the entire system, and once that's gone the rules stop being followed regardless of what the SOP says.

**Brand attribution is set at capture, never inferred.** The `product` field is populated from UTM parameters on the way in. This one comes from running three brands out of a single shared portal: it isn't hard to put multiple brands in one instance, it's hard to stop them bleeding into each other. That comes down to picking the one field that carries brand identity and guaranteeing it's populated at the form — because anything you infer downstream, you stop trusting.

**Leads that don't qualify still get a home.** The nurture branch assigns a track by source and calculates how many engagement minutes short the lead fell. Leads that fall out of routing with nothing assigned aren't lost, they're invisible, which is worse.

## Run it

1. Install n8n — `npx n8n` for a local instance at `http://localhost:5678`, or use n8n Cloud.
2. In the editor: **Workflows → Import from File** → `workflows/lead-capture-qualify-route.json`.
3. Click **Execute workflow** to arm the test webhook, then POST one of the examples:

```bash
curl -X POST http://localhost:5678/webhook-test/lead-capture \
  -H 'Content-Type: application/json' \
  -d @examples/01-enterprise-demo-request.json
```

The four examples cover the paths worth checking:

| File | What it exercises |
|---|---|
| `01-enterprise-demo-request.json` | High-value event qualifies at 12 minutes; routes to BDR |
| `02-mid-market-threshold.json` | 118 minutes crosses the threshold; routes to ISR |
| `03-event-below-threshold.json` | 20 minutes; nurture, 80 minutes short |
| `04-named-account-existing-owner.json` | Existing owner beats a 90-person headcount |

Example 01 also has a deliberately messy email (` Dana@BigCo.com `) to show normalization doing its job.

## What is deliberately not here

`Build Sales Handoff` constructs the Slack and SalesLoft payloads but doesn't send them. That keeps the repo runnable by anyone with no credentials to set up, and it keeps the reviewable part — the decision layer — in focus. To make it live, replace that node with two HTTP Request nodes pointed at a Slack incoming webhook and the SalesLoft API.

Also out of scope: enrichment, dedupe against an existing CRM, and the rolling three-month engagement calculation, which in production is a HubSpot property rather than something computed here. Each is a real part of the system and each would obscure the part this repo is meant to show.

## License

MIT
