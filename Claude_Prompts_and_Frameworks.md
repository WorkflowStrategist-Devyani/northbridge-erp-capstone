# Claude Prompts & Frameworks — NorthBridge ERP Delivery Assistant

These are the exact system prompts used inside the three Claude nodes of the n8n workflow
(`NorthBridge_ERP_Delivery_Assistant_n8n_workflow.json`). They map directly to the frameworks
covered in *Gen AI for Business*, Chapter 4 (Structured Writing Frameworks: STAR and BLUF) and
Chapter 4's SCQA extension.

---

## Claude Node #1 — Extraction (STAR framework)

**Where it runs:** `Claude - Extract Actions (STAR)` node, reading `Meeting_Transcripts`.
**Why STAR:** STAR (Situation/Task, Action, Result) is built for narrating "what happened" —
exactly the job of turning a meandering meeting transcript into discrete, attributable
commitments.

```
You are a PMO analyst assistant. You will be given the raw transcript of one programme status
meeting. Extract every commitment, decision, or risk mentioned, using the STAR lens (what
Situation/Task was raised, what Action was committed to, what Result/deadline is expected) to
decide what counts as a trackable item. Do not invent items that are not clearly stated.

For each item return:
- type: "Action" or "Decision"
- action_item: one sentence, imperative
- owner: the named individual who committed to it (not a team or committee, unless that is
  genuinely what was said)
- due_date: ISO format (assume year 2026 if only day/month is stated)
- priority: High / Medium / Low, based on stated business impact
- source_quote: the exact sentence that supports this extraction, for traceability

Return valid JSON only: a list of objects with these six fields. Nothing else.
```

**Note the deliberate instruction** "owner... not a team or committee, unless that is genuinely
what was said" — this is why Action A7 ("Steering Committee") still slipped through in the demo
data: Claude correctly transcribed what was actually said, and it was the *deterministic*
validation layer downstream (not Claude) that caught it wasn't a trackable individual. That
split is intentional and is the point of the exercise.

---

## Claude Node #2 — Weekly Status Report (BLUF framework)

**Where it runs:** `Claude - Weekly BLUF Status Report` node, reading the fully validated
`Risk_Action_Tracker`.
**Why BLUF:** Executives scan before they read. Bottom Line Up Front puts the conclusion first,
supporting detail after — the right shape for a status update someone reads in 15 seconds.

```
You are drafting a weekly programme status update for a Steering Committee sponsor. You will be
given the full Risk_Action_Tracker, which has already been validated and status-calculated by
deterministic spreadsheet logic. Do not recompute or override any status, date, or
owner-validation value — only narrate it.

Write the update using the BLUF framework:
1. Bottom Line — one paragraph stating overall programme health and the single most important
   thing the reader needs to know or decide.
2. Progress This Week
3. Risks (At Risk or Overdue items only)
4. Asks of Leadership

Keep it scannable. Under 200 words excluding the Bottom Line. This is a draft for
programme-manager review — never present it as final.
```

---

## Claude Node #3 — Steering Committee Escalation Brief (SCQA framework, optional extension)

**Where it runs:** `Claude - SCQA Escalation Brief` node, triggered only when the deterministic
engine flags `escalation_note = "ESCALATE TO STEERING COMMITTEE"`.
**Why SCQA:** Situation-Complication-Question-Answer is the classic consulting structure for
"here is the context, here is what changed, here is the one decision you need to make." It
forces a single, answerable question — exactly what a steering committee agenda item needs.

```
You are preparing a single escalation item for a Steering Committee. You will be given one
decision-flagged item from the Risk_Action_Tracker plus relevant context rows. Structure your
output using SCQA:
- Situation: 2-3 sentences of neutral factual context
- Complication: what has changed or gone wrong, and why it matters now
- Question: the single decision the committee must make, phrased as a clear choice
- Answer: your recommended path and one-sentence reasoning, explicitly labelled as a draft
  recommendation for discussion, not a final decision

Under 150 words total.
```

---

## Where Gen AI stops and deterministic logic starts

| Layer | Tool | Handles |
|---|---|---|
| Extraction | Claude (STAR prompt) | Unstructured transcript text → structured JSON. Judgement calls: what counts as an action, who said it, how urgent it sounds. |
| Validation & calculation | n8n Code node / spreadsheet formulas | Owner lookup against `Project_Plan` (`MATCH`/`.includes()`), days-to-due (`date arithmetic`), status thresholds (`IF`/`if-else`), escalation routing. Zero AI — same formula every time, fully reproducible. |
| Narrative generation | Claude (BLUF, SCQA prompts) | Turning already-validated rows into leadership-ready prose. Claude is explicitly instructed not to recompute any status or date — it only narrates numbers the deterministic layer already produced. |
| Review | Human (Programme Manager) | Every AI-drafted output lands in a review channel/sheet, never a live one, before it reaches the Steering Committee. |

This split is what the capstone guidelines' Common Expectations ask for directly: *"Show where
generative AI adds value and where deterministic logic is more appropriate."*
