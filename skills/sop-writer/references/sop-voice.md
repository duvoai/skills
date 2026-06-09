# SOP Voice

These rules govern SOP body text — the operational instructions an Assignment executes at Run-runtime. SOPs are stricter than general product copy: the audience is an LLM driving real Connections, not a person reading the UI.

## Outcome-focused

Write the **business result**, not the mechanical action.

| ❌ Don't                       | ✅ Do                                |
| ------------------------------ | ------------------------------------ |
| "Call the `create_order` API." | "Place the order with the supplier." |
| "POST to `/v1/invoices`."      | "Post the invoice to NetSuite."      |
| "Run a `match_po` query."      | "Find the matching purchase order."  |

The Assignment knows how to translate "place the order" into the right tool call. Don't pre-decide tool calls in the SOP body.

The single exception is the **case lifecycle tools** (`claim_case`, `update_case`, `postpone_case`, `complete_case`, `fail_case`, `add_cases`, `list_cases`, `request_handover`). These are platform primitives whose names matter to the runtime — reference them by name in the step where they apply. See `case-lifecycle.md`.

## Human-readable to a non-engineer

The reader of last resort is an operations manager, not a developer. If a sentence requires technical context to parse, rewrite it.

- "API call" → "request"
- "endpoint" → leave it out
- "webhook" → "notification"
- "the response object" → "the result"

## Duvo terminology

Use Duvo's nouns. Never substitute.

| Use        | Not                            |
| ---------- | ------------------------------ |
| Assignment | agent, AI teammate, bot        |
| Run        | task, job, execution           |
| SOP        | instructions, prompt, playbook |
| Connection | integration, account           |
| Files      | knowledge base, documents      |
| Login      | credential, password           |
| Start Work | run agent, execute             |
| Setup      | configuration, config          |

## Reference Connections by name

When a step uses a Connection, name it. This is how the runtime Assignment knows which tool to call.

| ❌ Don't                     | ✅ Do                                          |
| ---------------------------- | ---------------------------------------------- |
| "Send an email to the team." | "Use **Gmail** to send an email to the team."  |
| "Check the spreadsheet."     | "Search **Google Sheets** for the supplier."   |
| "Look up the supplier."      | "Use **Snowflake** to query supplier history." |

Bolding the Connection name is a useful convention but not required. The important thing is that the Connection appears in the step that uses it.

## Short paragraphs and steps

No walls of text. Two to four sentences per paragraph; one to four sub-bullets per step. If a step exceeds that, split it or move detail into sub-bullets. If a SOP section runs longer than a screen, it's doing too much — break the step apart.

Aim for 4–10 top-level numbered steps in `# STEPS`. More than that and the Assignment scope is probably too broad — see the decomposition signal in `SKILL.md`.

## Imperative voice for steps

Steps are commands, not descriptions.

| ❌ Don't                                               | ✅ Do                         |
| ------------------------------------------------------ | ----------------------------- |
| "The invoice should be matched to a PO."               | "Match the invoice to a PO."  |
| "The Assignment will then check the vendor whitelist." | "Check the vendor whitelist." |

## Concrete over vague

Replace fuzzy adverbs with concrete thresholds, even at the cost of specificity. The user can edit a number; they cannot edit "appropriate".

| ❌ Don't                              | ✅ Do                                  |
| ------------------------------------- | -------------------------------------- |
| "Escalate if the amount is large."    | "Escalate if the amount exceeds $500." |
| "Wait a reasonable time for a reply." | "Wait 24 hours for a reply."           |
| "Handle the request appropriately."   | (rewrite — this is not a step)         |

## Inline conditions with `If [condition]: [action]`

Decision criteria, early returns, and error handling all use the same shape. Put them as sub-bullets under the step where the condition is evaluated.

| ❌ Don't                                                          | ✅ Do                                                                                               |
| ----------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| Separate `# ERROR HANDLING` section listing every possible error. | Inline: "If the supplier is not on the whitelist: call `fail_case` with reason 'supplier-unknown'." |
| "Use your judgment to decide whether to escalate."                | "If the line discrepancy exceeds 5% or $500: use **Human in the loop** before posting the bill."    |

## No commented-out alternatives

Don't write "consider X or maybe Y". Pick one. The SOP is operational; the Assignment will follow whatever you wrote. If a choice is genuinely up to the operator, capture it as a Setup input the user fills in, not as an inline alternative.
