---
name: codex-query-leads
description: Use when the user wants to query, filter, or look up leads in the Codex CRM — e.g. "show me all qualified leads from LinkedIn", "find leads at Acme", "how many contacted leads do we have this month", "who are my newest unqualified leads". Read-only — never writes.
---

# Query leads in the Codex CRM

This skill reads from the Codex leads database via its REST API. It never mutates.

## Required configuration

Environment variables:
- `CODEX_API_URL` — base URL of the backend
- `CODEX_USER_EMAIL` — for the `X-User-Email` header

If missing, ask the user before proceeding.

## Endpoint

```
GET $CODEX_API_URL/api/v1/leads
```

### Useful query parameters

| Param                  | Notes                                                                    |
|------------------------|--------------------------------------------------------------------------|
| `page`, `limit`        | Pagination (default page=1, limit=20, max=100)                           |
| `search`               | Free-text across name, email, and company                                |
| `status`               | `new` / `contacted` / `qualified` / `unqualified` / `converted` / `lost` |
| `emailStatus`          | `none` / `sent` / `replied` / `bounced`                                  |
| `emailVerifiedStatus`  | `valid` / `invalid` / `catch_all` / `unknown`                            |
| `enrichmentStatus`     | `pending` / `success` / `failed`                                         |
| `company`              | Substring match, case-insensitive                                        |
| `jobTitle`             | Substring match, case-insensitive                                        |
| `seniority`            | Exact match                                                              |
| `country`              | Substring match                                                          |
| `sourceType`           | Exact match (`linkedin`, `apollo`, `csv`, custom slugs from settings)    |
| `listId`               | Only leads in a given list                                               |
| `tagId`                | Only leads with this tag                                                 |
| `ownerId`              | Only leads owned by this user                                            |
| `hasEmail`             | `true` / `false`                                                         |
| `hasLinkedinUrl`       | `true` / `false`                                                         |
| `duplicateFlag`        | `true` / `false`                                                         |

## Workflow

1. **Translate the user's question into query parameters.** For example:
   - "qualified leads at Acme" → `status=qualified&company=Acme`
   - "leads from LinkedIn this month" → `sourceType=linkedin` + filter the response by `createdAt` client-side (the API does not currently support a date range param on this endpoint)
   - "John Doe" → `search=John Doe`

2. **Make the request:**

   ```bash
   curl -sS -G "$CODEX_API_URL/api/v1/leads" \
     -H "X-User-Email: $CODEX_USER_EMAIL" \
     --data-urlencode "status=qualified" \
     --data-urlencode "company=Acme" \
     --data-urlencode "limit=50"
   ```

3. **Parse the response.** Shape:

   ```json
   {
     "success": true,
     "data": [ /* leads */ ],
     "meta": { "page": 1, "limit": 50, "total": 12, "totalPages": 1 }
   }
   ```

4. **Summarize for the user.** Default to a compact table (name, company, title, status, owner). For aggregate questions ("how many"), report the count from `meta.total` rather than listing every row. For very large result sets, show the first 10 + the total and offer to refine the query.

## Single-lead lookup

If the user already has a lead id, fetch the full record with relations:

```bash
curl -sS "$CODEX_API_URL/api/v1/leads/<id>" \
  -H "X-User-Email: $CODEX_USER_EMAIL"
```

This response includes the lead's `tags`, `owner`, `listEntries`, and enrichment data.

## What you cannot do here

- This skill **does not** create, update, or delete leads. If the user asks for those, switch to `codex-add-lead` or `codex-update-lead-status`.
- Date-range filtering must be done client-side on the response — the list endpoint doesn't yet expose `createdAt` filters.
