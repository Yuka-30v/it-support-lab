# API Error Triage Assistant

A Claude Project that helps a support agent work through an API error: what it means, what to check first, a customer reply and an escalation note. Built and tested in October 2026.

## The problem

When a customer reports an API error, a support agent has to do four things quickly and without gaps: understand the error, narrow down the cause, reply to the customer, and, if needed, escalate with enough evidence that the next team does not repeat the investigation.

While building my [HubSpot CRM API Lab](../hubspot-crm-api-lab/README.md), I saw an AI assistant get two of five errors wrong. It assumed an OAuth token when the key was a service key, and it blamed a duplicate email for what was actually a misspelled property name. I wanted an assistant that reads the raw error, says what it does not know, and keeps a consistent structure an agent can trust.

## Design

**Tool:** Claude Project (project instructions plus one reference file).

**Input:** an error response (text or screenshot) and, if known, what action caused it.

**Output:** the same four sections every time.

| Section | Why it is there |
|---|---|
| 1. What this error means | Plain language, up to 5 lines, so a non-technical agent understands it before acting. |
| 2. Likely causes and what to check first | A fixed order so checks are not skipped: client error (4xx) or server error (5xx) first, then authentication, permissions, request details and whether the object exists. |
| 3. Customer reply draft | What is known, what is still being investigated, and when the customer will hear back. |
| 4. Escalation note | Customer impact, steps to reproduce, what has been tested, the error ID and the help needed, so the next team can act without starting over. |

**Reference file:** the HubSpot lab README, which contains five confirmed cases. It is treated as the right answer for HubSpot errors and only as an example of reasoning for other APIs.

## Instructions (final version)

```
## Role
You help customer support staff investigate API errors. The user will paste an error response (text or screenshot) and, if known, what action caused it.

## Output format
Always answer with these four sections, in this order:
1. What this error means: up to 5 lines, in plain language a non-technical support agent can understand.
2. Likely causes and what to check first: First decide whether it is a client error (4xx) or a server error (5xx). For client errors, check in this order: authentication, permissions, request details, and whether the object exists. For server errors, explain that the cause is likely on the provider's side and focus on the escalation note.
3. Customer reply draft: what is known, what is still being investigated, and when the customer will hear back.
4. Escalation note: customer impact, steps to reproduce, what has been tested, the error ID (such as correlationId), and the help needed.

## Rules
- Do not assume the authentication method (for example, OAuth) unless the evidence shows it.
- If key information is missing, ask for it before answering. If you must guess, label it as a guess.
- If a screenshot shows an API key or token, warn the user first and tell them to rotate it.
- Use only the four sections above. Put any extra observations as short bullets at the end of section 2.
- Facts from the README are not evidence about the current case. Only treat something as confirmed if it appears in the user's input. Otherwise, label it as an assumption.
- Keep extra observations to 3 short bullets. Do not comment on how the user's test environment is set up.
- In the customer reply, only describe actions that have already happened. Write planned actions as "we will".
- Do not tell the customer that something is ruled out unless the input proves it.

## Reference
The HubSpot lab README contains confirmed cases. For HubSpot errors, treat it as the right answer. For other APIs, use it only as an example of how to reason.
```

The first version had only the first three rules. Each later rule was added because a specific test failed (see below).

## Testing

I ran each test in a new chat so earlier answers could not influence the result. When a test failed, I changed one thing in the instructions and ran the same input again.

| # | Input | Result | What I changed |
|---|---|---|---|
| 1 | 403 screenshot from my HubSpot lab | Correct diagnosis, guesses labelled, token check done. **Failed format:** added 3 to 4 extra sections. | Rule: only four sections, extras at the end of section 2. |
| 1b | Same 403 | Four sections only. Left placeholders instead of inventing missing data. Judged that escalation was not needed unless the fix failed. **Pass.** | None |
| 2 | 404 screenshot (HTML body, not JSON) | Spotted the HTML response and the risk that an integration parsing errors as JSON could crash. **Failed:** stated facts from the README as "tested" in this case. | Rule: README facts are not evidence about the current case. Rule: at most 3 extra bullets, no comments on the test setup. |
| 2b | Same 404 | Authentication labelled as an assumption. "Tested so far" limited to what the screenshot showed. Noted that a wrong host or path can also return an HTML 404. One unlabelled assumption remained. **Mostly pass.** | None |
| 3 | Only the text "I got a 401 error" | Asked five targeted questions before answering, including whether anything changed recently. Asked me to blur keys before pasting. **Pass.** | None |
| 4 | 503 with a correlationId (practice input) | Treated it as a server error, skipped client-side checks, put the correlationId in the escalation note. **Failed in the customer reply:** said the issue had already been escalated, and told the customer their credentials were fine without evidence. | Rules: describe only completed actions in the reply; do not rule things out without proof. |
| 4b | Same 503 | Reply used "we will" for planned actions and no longer ruled out credentials. Added a check for duplicates before retrying failed creates. **Pass.** | None |
| 5 | Stripe 401 with a masked live key (practice input) | No HubSpot assumptions; Stripe-specific checks (rolled keys, Dashboard, Request-Id). Recognised the live key and raised priority. **Pass.** | None |

## What I learned

- **A rule only works where it is aimed.** The "label assumptions" rule fixed the escalation note in Test 2b, but the same mistake came back in the customer reply in Test 4. The reply needed its own rules.
- **The most dangerous failure was quiet.** In Test 2 the assistant listed checks as "tested" that nobody had run. An escalation note like that would send a specialist team down the wrong path. It looked confident and well formatted, which made it easy to miss.
- **The assistant sometimes judged better than I did.** In Test 5 I expected a warning about the partly visible Stripe key. Instead it explained that a masked key with only the last four characters is safe to share and told the customer not to send the full key. That was the better call, and I kept the rule as it was.
- **Account settings can leak into a project.** In Test 1 the assistant added a "weak point to consider" section I never asked for. This most likely came from my personal Claude preferences, which ask for critical feedback. A tool meant for other people should not depend on one person's account settings.

## Limitations

- Tested with eight runs, mostly on HubSpot. Other providers were tested once (Stripe).
- Small unlabelled assumptions can still appear, for example in reproduction steps (Test 2b).
- In Test 5 the escalation note said questions "have been sent" before the reply was sent. The "completed actions only" rule covers the customer reply but not the escalation note yet.
- Practice inputs for Tests 4 and 5 were written for testing, not taken from real customer cases.
- Every output is a draft. An agent should read and check it before sending anything to a customer.
