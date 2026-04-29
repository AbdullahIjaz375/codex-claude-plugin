---
name: codex-add-lead
description: Use when the user wants to add a new lead to the Codex CRM database — e.g. "add John Doe at Acme as a new lead", "create a lead from this LinkedIn URL", "save this contact to Codex". Performs deduplication automatically (matches on email, LinkedIn URL, or name+company). Always shows a summary and asks for confirmation before writing.
---

# Add a lead to the Codex CRM

This skill creates a new lead in the Codex CRM via its REST API.

## Required configuration

The user's environment must define:
- `CODEX_API_URL` — base URL of the backend (e.g. `http://localhost:3001`)
- `CODEX_USER_EMAIL` — the email of the user making the request (used for `X-User-Email` auth header)

If either is missing, ask the user for the value before proceeding.

## Workflow

1. **Gather lead details** from the user. The most useful fields:
   - `firstName`, `lastName` (or `email`, or `linkedinUrl`) — at least one of these is required for a meaningful record
   - `email`, `phone`, `linkedinUrl`, `company`, `jobTitle`, `seniority`, `department`, `location`, `country`, `notes`, `sourceType`

2. **Show a summary table** of what you are about to create. Example:

   ```
   I'm about to create this lead in Codex:

   | Field      | Value                               |
   |------------|-------------------------------------|
   | First name | John                                |
   | Last name  | Doe                                 |
   | Email      | john@acme.com                       |
   | Company    | Acme                                |
   | Job title  | VP Engineering                      |
   | Source     | linkedin                            |

   Confirm? (yes/no)
   ```

3. **Wait for explicit confirmation.** Do not call the API until the user says "yes" or equivalent. If they want changes, update the summary and re-confirm.

4. **Call the API** using `Bash` + `curl`:

   ```bash
   curl -sS -X POST "$CODEX_API_URL/api/v1/leads" \
     -H "Content-Type: application/json" \
     -H "X-User-Email: $CODEX_USER_EMAIL" \
     -d '{
       "firstName": "John",
       "lastName": "Doe",
       "email": "john@acme.com",
       "company": "Acme",
       "jobTitle": "VP Engineering",
       "sourceType": "linkedin"
     }'
   ```

5. **Report the result.** The API returns `{ success, data: { id, ... } }`. Tell the user the lead's id and confirm the create succeeded. If the email or LinkedIn URL matched an existing lead, the response is the merged record — note that to the user explicitly ("Existing lead matched and updated").

## Deduplication

The backend matches in this priority order: LinkedIn URL → email → firstName+lastName+company. If any matches, the lead is **updated** rather than created. Tell the user which case happened.

## Errors

- `400` with `error` field — show the validation message.
- `401`/`403` — the `CODEX_USER_EMAIL` is wrong or the user is not authorized. Ask the user to fix the env var.
- Network failure — show the error and stop.
