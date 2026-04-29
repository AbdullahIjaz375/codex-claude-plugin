---
name: codex-update-lead
description: Use when the user wants to change ANY field on one or more leads in the Codex CRM — pipeline status, owner, tags, name, email, phone, company, job title, location, notes, anything editable. Examples — "mark John Doe as qualified", "change Sara's title to VP of Eng", "set the owner of these leads to Mike", "add the YC tag to all Series A leads from Apollo", "update the email for lead X to ...". MANDATORY confirmation before any write — always show the proposed change and the affected leads first.
---

# Update lead fields in Codex

This skill performs **any** mutation on Codex leads — single-field edits, bulk operations, status changes, owner reassignments, tag additions, anything. Confirmation is mandatory because changes affect downstream campaigns, reporting, and audit trails.

## Required configuration

Environment variables:
- `CODEX_API_URL` — base URL of the backend (e.g. `https://codex.gocodeable.com`)
- `CODEX_USER_EMAIL` — the user's email for the `X-User-Email` audit header

If either is missing, ask before proceeding.

## Editable fields

Any of these can be set on a single lead via `PUT /api/v1/leads/:id`:

| Field | Type / values |
|---|---|
| `firstName`, `lastName` | string |
| `email` | valid email or empty string |
| `phone` | string |
| `linkedinUrl` | URL string |
| `profileSummary` | string (longer) |
| `company`, `companyWebsite`, `companyLinkedinUrl`, `companyDescription`, `companyHeadcount`, `companyIndustries`, `companyAddress` | string |
| `jobTitle`, `jobDescription`, `department`, `seniority` | string |
| `country`, `location` | string |
| `notes` | string |
| `sourceType` | string (must match a SourceType slug like `apollo`, `linkedin`, `csv`, `referral`, etc.) |
| `status` | one of `new` / `contacted` / `qualified` / `unqualified` / `converted` / `lost` |
| `ownerId` | a User id, or `null` to unassign |

For **bulk** operations, `POST /api/v1/leads/bulk/update` accepts: `leadIds` (array, required), `status`, `ownerId`, `tagIds` (array of label ids).

For **tag operations** specifically, use `POST /api/v1/leads/:id/tags` (body `{ tagIds: ["..."] }`) to add and `DELETE /api/v1/leads/:id/tags` (body `{ tagIds: ["..."] }`) to remove.

## Workflow — single lead, single field

1. **Resolve the lead.** If the user gave a name, search via `GET /api/v1/leads?search=<name>` to find the lead id. If multiple match, show the candidates and ask which one.

2. **Show the proposed change.** Example:

   ```
   I'll update this lead:

   - Lead: John Doe <john@acme.com> (id: clxx...)
   - Field: jobTitle
   - Current: "Senior Engineer"
   - New:     "VP of Engineering"

   Confirm? (yes/no)
   ```

3. **Wait for explicit "yes".** Then call:

   ```bash
   curl -sS -X PUT "$CODEX_API_URL/api/v1/leads/<id>" \
     -H "Content-Type: application/json" \
     -H "X-User-Email: $CODEX_USER_EMAIL" \
     -d '{ "jobTitle": "VP of Engineering" }'
   ```

4. **Report the result.** Echo the new value from the response so the user sees the write took.

## Workflow — single lead, multiple fields at once

Combine the fields in one request body. Always show the full diff in the summary before confirming.

```bash
curl -sS -X PUT "$CODEX_API_URL/api/v1/leads/<id>" \
  -H "Content-Type: application/json" \
  -H "X-User-Email: $CODEX_USER_EMAIL" \
  -d '{ "jobTitle": "VP of Engineering", "company": "Acme", "status": "qualified" }'
```

## Workflow — bulk update

1. **Identify the lead set.** Either the user gave specific ids, or you need to query `GET /api/v1/leads?<filters>` to materialize the list. Page through if `meta.total` exceeds your first page.

2. **Show a summary** with the *count*, the *current value distribution*, and the *target value*. Example:

   ```
   I'll update 47 leads:

   - From list: "Apollo Q1 Fintech CEOs"
   - Field: status
   - Current distribution: 30× new, 12× contacted, 5× qualified
   - New value: contacted
   - First few: John Doe, Jane Smith, Bob Lee, ... (show up to 10)

   Confirm? (yes/no)
   ```

3. **Wait for explicit "yes".** For status / owner / tag bulk updates, use the bulk endpoint:

   ```bash
   curl -sS -X POST "$CODEX_API_URL/api/v1/leads/bulk/update" \
     -H "Content-Type: application/json" \
     -H "X-User-Email: $CODEX_USER_EMAIL" \
     -d '{
       "leadIds": ["id1", "id2", "..."],
       "status": "contacted"
     }'
   ```

   For other fields (no bulk endpoint exists yet), iterate `PUT /api/v1/leads/:id` one at a time. Warn the user that this is sequential and will take a while if the set is large.

4. **Report the count updated.**

## Workflow — adding / removing tags

```bash
# Add tags
curl -sS -X POST "$CODEX_API_URL/api/v1/leads/<id>/tags" \
  -H "Content-Type: application/json" \
  -H "X-User-Email: $CODEX_USER_EMAIL" \
  -d '{ "tagIds": ["tagId1", "tagId2"] }'

# Remove tags
curl -sS -X DELETE "$CODEX_API_URL/api/v1/leads/<id>/tags" \
  -H "Content-Type: application/json" \
  -H "X-User-Email: $CODEX_USER_EMAIL" \
  -d '{ "tagIds": ["tagId1"] }'
```

If the user gave tag *names*, look up ids via `GET /api/v1/lists/labels/all` first.

## Safeguards — non-negotiable

- **Never** mutate without an explicit "yes" from the user in the conversation. "Sounds good", "go ahead", "do it" all count; ambiguous responses do not.
- For bulk updates affecting **more than 100 leads**, ask twice: once for the change itself, then a second confirmation that includes the affected count ("This will change 1,247 leads — confirm?").
- For status `lost` or `unqualified`, ask whether to add a reason as `notes` — if yes, include it in the update body alongside the status.
- For changes to `email`, **never** silently overwrite a different existing value; flag the change in the summary explicitly. Email is the dedup key downstream.
- For changes to `ownerId`, confirm the new owner exists by checking `GET /api/v1/settings/users` if you're not certain of the id.

## Errors

- `404` — the lead id wasn't found; offer to search again.
- `400` with a Zod validation error — show the message verbatim; the user usually needs to pick a different value.
- `401` / `403` — `CODEX_USER_EMAIL` is wrong or the user lacks permission. Ask them to fix the env var.
- Network failure — surface and stop.
