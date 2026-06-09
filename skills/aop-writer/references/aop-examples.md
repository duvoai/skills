# AOP Examples

Three canonical, full-text AOPs. Use them as calibration anchors before returning a draft — your output should read like one of these.

The examples are progressively more complex:

1. **Standalone deep AOP** — a single Agent that does its whole job in one Run, no queues.
2. **Queue consumer with Human-in-the-Loop** — a consumer AOP with HITL gating and terminal closure.
3. **Postpone-driven reminder step** — a producer+consumer AOP using the postpone-then-retry idiom.

---

## Example 1 — Standalone deep AOP

A single Agent, triggered on a schedule (configured in Setup, outside the AOP). Runs end to end on each invocation. No queue, no handover.

```markdown
# GOAL

Generate the daily revenue summary for yesterday and post it to the leadership Slack channel.

# STEPS

1. Determine the report window: yesterday in the team's timezone (Europe/Prague), 00:00 to 23:59:59.

2. Use **Snowflake** to query revenue for the window:
   - Pull total revenue, revenue by product line, and the top 10 customers by revenue.
   - If the query returns zero rows: post a message in **Slack** saying "No revenue recorded yesterday — please verify the data pipeline." and stop.

3. Use **Web Search** to pull the headline FX rate for USD/EUR at market close yesterday, for context on the EUR-denominated lines.

4. Format the summary as a Slack message:
   - Headline: total revenue, day-over-day change in %, week-over-week change in %.
   - Product-line table with revenue and DoD %.
   - Top 10 customers as a bulleted list with revenue.
   - Footer: "Data as of [timestamp], FX rate USD/EUR: [rate]."

5. Post the message to the `#leadership-daily` channel via **Slack**.
   - If posting fails: retry once after 30 seconds. If it still fails, post to `#ops-alerts` with the formatted summary and the error reason.

6. Log the run in **Google Sheets**, appending a row to the "Daily Revenue Reports" sheet: date, total revenue, DoD %, WoW %, posting status.

# NOTES

- All times in the AOP refer to Europe/Prague unless explicitly UTC.
- The "Daily Revenue Reports" sheet must only be appended to, never overwritten.
```

What makes this excellent:

- One sentence GOAL, single-day framing.
- Connection names appear in every step that uses one.
- Inline early returns ("If the query returns zero rows…") instead of a separate ERROR HANDLING section.
- Concrete fallback for Slack posting failure.
- `# NOTES` carries genuinely cross-cutting facts.

---

## Example 2 — Queue consumer with Human-in-the-Loop

The Agent is a queue consumer with the `case-queue-consumer` Connection. Each Run claims one case from the "Pending Refunds" queue and processes it end to end.

```markdown
# GOAL

Process one refund request case end to end: validate it, approve or escalate, and either issue the refund or close the case with a reason.

# STEPS

1. Call `claim_case` to get the case data. The case carries: order ID, customer email, refund amount, reason code, and submitted-by timestamp.

2. Use **Shopify** to look up the order:
   - If the order is not found: call `update_case` to record `error: order-not-found`, then `fail_case`. Stop.
   - If the order is older than 90 days: call `update_case` to record the age, then `fail_case` with reason "outside-return-window". Stop.

3. Check refund eligibility:
   - If the refund amount exceeds the original order total: cap at the order total and note the adjustment in the case data via `update_case`.
   - If the reason code is one of (`damaged`, `wrong-item`, `not-as-described`): mark as "auto-approve" candidate.
   - Otherwise: mark as "review-required".

4. If the candidate is "auto-approve" AND the (capped) refund amount is under $200:
   - Use **Stripe** to issue the refund against the order's payment method.
   - Use **Gmail** to send the customer a confirmation email with the refund amount and the original order link.
   - Call `update_case` with `outcome: auto-approved-refunded` and `complete_case`. Stop.

5. Otherwise (review-required OR amount ≥ $200):
   - Use **Human in the loop** with a summary: customer email, order ID, refund amount, reason code, and order age. Ask: "Approve refund? (yes / no / request-info)".
   - If the human responds "yes": issue the refund via **Stripe**, send the confirmation via **Gmail**, call `update_case` with `outcome: human-approved-refunded`, then `complete_case`.
   - If the human responds "no": use **Gmail** to send the customer a decline email referencing the human's reason note. Call `update_case` with `outcome: human-declined` and `complete_case`.
   - If the human responds "request-info": use **Gmail** to send the customer the human's question. Call `update_case` to set `awaiting_customer_info: true` and `postpone_case` with `postpone_to: "3d"` to follow up.
```

What makes this excellent:

- Step 1 is `claim_case` — non-negotiable for consumer AOPs.
- Every branch reaches a terminal call (`complete_case`, `fail_case`, or `postpone_case`).
- HITL is a first-class step with a concrete prompt, not "use your judgment".
- Concrete thresholds ($200, 90 days) — no fuzzy adverbs.

---

## Example 3 — Postpone-driven reminder step

Excerpt from a producer+consumer Agent in a reminder chain. The Agent claims a case from the "First Reminder" queue, waits 1 day on first pickup, then sends the reminder and advances the case to the next queue.

```markdown
# GOAL

Send the first reminder for one outstanding purchase order one day after the case lands in the queue, then advance it to the second-reminder queue.

# STEPS

1. Call `claim_case` to get the case data. The case carries: PO number, supplier name, responsible buyer email, original order date, and Slack thread URL.

2. If `initial_postpone_done` is NOT present in the case data (this is the first pickup):
   - Call `update_case` to set `initial_postpone_done: true` in the case data.
   - Call `postpone_case` with `postpone_to: "1d"`.
   - Stop — do not continue.

3. (Second pickup — `initial_postpone_done` is true.) Use **Slack** to read the original PO thread and check whether the supplier has replied since the case was opened:
   - If a reply exists: call `update_case` with `outcome: supplier-replied-before-reminder` and `complete_case`. Stop — no reminder needed.

4. Use **Gmail** to send a first-reminder email to the supplier:
   - Subject: "Reminder: PO [PO number] awaiting response".
   - Body references the original order date and the Slack thread link.
   - CC the responsible buyer.

5. Post a short note in the original **Slack** thread: "Sent first reminder to [supplier name]."

6. Call `add_cases` on the "Second Reminder" queue to enqueue this PO for follow-up:
   - `title`: "Second reminder: PO [PO number]".
   - `data`: copy the original case data plus `first_reminder_sent_at: <now>`.

7. Call `update_case` with `outcome: first-reminder-sent`, then `complete_case`.
```

What makes this excellent:

- Step 1 is `claim_case`.
- Step 2 is the canonical postpone-then-retry idiom — a flag on the case, then `postpone_case`, then exit.
- Step 3 handles the "no reminder needed" branch with a terminal `complete_case`.
- The producer side (`add_cases` to the next queue) is wired in step 6 before the consumer-side `complete_case` in step 7.
- Every branch terminates.

---

## What these examples do not show

- **No `# PREREQUISITES`** section — the Connections used appear in the steps that need them.
- **No `# ESCALATION RULES`** section — escalation lives inline as `If [condition]: use Human in the loop / request_handover / fail_case …`.
- **No `# SUCCESS CRITERIA`** section — the GOAL line states the outcome; the terminal step in each branch demonstrates it.
- **No batch loops** — each example processes one item.
- **No "use your judgment"** — every decision has a concrete threshold or rule.

Match this calibration before returning a draft.
