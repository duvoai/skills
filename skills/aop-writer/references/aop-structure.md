# AOP Structure

Every Duvo AOP uses the same three-section markdown shape. Do not invent new top-level sections. Do not split the AOP into separate `PREREQUISITES` / `ERROR HANDLING` / `DATA HANDLING` / `ESCALATION` / `TRIGGER` sections — those concerns are inlined into the relevant step.

## The three sections

### `# GOAL`

One sentence describing what the Agent does on a single case and the expected outcome, stated as a business result. No process language.

> # GOAL
>
> Process one supplier negotiation case by researching the supplier, evaluating current terms, and either completing the negotiation or escalating to a human.

If the Agent is queue-driven, name the kind of case it processes — "one supplier negotiation case", "one inbound expense report", "one DESADV exception". Single-case framing is the most important property of the GOAL line.

### `# STEPS`

A numbered list of imperative actions. Each step is a short imperative sentence describing the **business action**, with everything that belongs to that step inlined as sub-bullets: data extracted, decision criteria, early returns, error handling, terminal action.

Aim for **4–10 top-level steps**. If you need more, the Agent scope is probably too broad — see the decomposition signal in `SKILL.md`.

> # STEPS
>
> 1. Read the case details: supplier name, current contract terms, requested changes, and priority level.
>    - If priority is "low" and the requested discount is under 2%: mark as "auto-approved" and stop — no negotiation needed.
> 2. Use **Web Search** to research the supplier:
>    - Look for recent financial news, market position, and competitor pricing.
>    - Check for any supply chain disruptions or risks.
> 3. Use **Enterprise Browser** to log into the procurement system and pull the supplier's full history: past orders, payment terms, dispute history, and current contract expiration date.
> 4. Evaluate the negotiation position:
>    - If the supplier has a history of late deliveries (>3 in the past year): note this as leverage.
>    - If we represent >20% of their revenue (check order volumes): note strong negotiating position.
>    - If the contract expires within 30 days: flag as urgent.
>    - If we have no leverage and the requested terms are within market range (±5% of Web Search benchmarks): approve as-is, mark as "approved-as-requested", and stop.
> 5. Draft a negotiation email via **Gmail**:
>    - Reference specific data points from research.
>    - Propose counter-terms based on evaluation.
>    - Set a response deadline of 5 business days.
> 6. Log the negotiation in **Google Sheets** by adding a row to the "Active Negotiations" sheet with: supplier name, proposed terms, research summary, priority, and date sent.
> 7. If priority is "critical" or contract value exceeds $100,000: use **Human in the loop** to get approval before sending the email. If rejected, revise per the human's feedback and re-draft.
> 8. Send the negotiation email, then call `update_case` to record the outcome ("negotiation-sent") in the case data and `complete_case` to mark the case done.

Step shape rules:

- **Imperative voice.** "Read the case details." — not "The Agent should read the case details."
- **Describe what, not how.** "Find the matching purchase order in NetSuite." — not "Call the `find_po` tool with parameters …". The Agent chooses the tool call.
- **Reference tools by Connection name.** "Use **Gmail** to send…" / "Search in **Google Sheets** for…". Bolding the Connection name is a useful signal but not required.
- **Inline decision criteria as sub-bullets** with concrete thresholds, not "use your judgment".
- **Inline early returns** with `If [condition]: [action] and stop.`
- **Blank line between top-level steps** for readability.

### `# NOTES` _(optional)_

Use only for cross-cutting concerns that don't belong to a single step. Examples: timezone conventions, naming patterns, a fact about the data that applies to every step.

> # NOTES
>
> - All timestamps in case data are stored in UTC; convert to Europe/Prague for display in Slack messages.
> - When writing back to Google Sheets, always preserve existing rows — never overwrite, only append.

Omit the section entirely if there are no such concerns. Don't pad it with content that should live in a specific step.

## What inlining replaces

| What you might be tempted to write as a section             | Where it goes instead                                                                                                                                      |
| ----------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `# PREREQUISITES` ("the Gmail Connection must be linked")   | The Agent's Setup; the AOP assumes Connections work. If a step might fail because a Connection is missing, put `If [tool] is unavailable: …` in that step. |
| `# ERROR HANDLING` ("if any step fails, log and notify")    | Inline `If [condition]: [action]` into the step where the failure can occur.                                                                               |
| `# ESCALATION RULES` ("escalate to AP clerk if …")          | Inline as an `If …: use Human in the loop / request_handover / fail_case with reason …` sub-bullet.                                                        |
| `# SUCCESS CRITERIA` ("the invoice is posted or escalated") | The GOAL line describes the expected outcome; the terminal step in each branch demonstrates the success criterion concretely.                              |
| `# TRIGGER` ("runs every Monday")                           | Trigger configuration lives outside the AOP. The AOP describes what to do once triggered, on one case.                                                     |

## Why this shape

The three-section structure is the canonical Duvo AOP shape and what the Duvo documentation teaches. Diverging from it creates friction for users iterating on existing Agents or learning from the docs.

The shape is deliberately flat: a clear single-sentence GOAL, then a numbered list of steps that each contain everything needed to execute them. Self-contained steps make the AOP easy to scan, easy to debug ("step 4 is where it went wrong"), and easy to interpret consistently at run time.
