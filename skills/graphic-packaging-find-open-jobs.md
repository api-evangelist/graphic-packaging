---
name: graphic-packaging-find-open-jobs
description: >-
  Search and read Graphic Packaging International's open job requisitions from
  the unauthenticated careers job query API, including the two gotchas that make
  naive calls return zero results.
generated: '2026-09-12'
method: generated
source: >-
  Grounded entirely in live probes of https://careers.graphicpkg.com/api/mcp/jobs
  and the published https://careers.graphicpkg.com/llms.txt on 2026-09-12. Every
  tool name, parameter name, response field and failure below was observed on a
  real response; nothing is inferred.
api: graphic-packaging-career-site-job-query-api
operations:
  - search_jobs
  - get_job
  - list_departments
  - list_locations
---

# Find open jobs at Graphic Packaging

**Base:** `https://careers.graphicpkg.com/api/mcp/jobs` — one endpoint, GET only,
no authentication, CORS open. The operation is chosen with a `tool` query
parameter, not a path.

**This is not an MCP server.** The `/api/mcp/` path is misleading. POSTing a
JSON-RPC `tools/list` returns `405`. Use plain GET.

**Whose API this is.** The surface is operated by the AppVault career-site
platform under a Graphic Packaging hostname, and the postings themselves live in
SAP SuccessFactors. It is not a Graphic Packaging product API and carries no
support commitment, no versioning and no published rate limit.

## 1. Search requisitions

```
GET /api/mcp/jobs?tool=search_jobs&search=press%20operator&pageSize=10
```

Optional parameters: `search` (keyword over title and description),
`department` (numeric id — see the gotcha below), `employmentType`,
`location` (city, state or zip), `page` (default 1), `pageSize` (default 10).

The response is `{tool, results[], totalCount, page, pageSize, summary}`. Each
result is a reduced projection: `requisitionId`, `title`, `department`,
`location` (a flattened "City, ST, USA" string), `employmentType`, `datePosted`,
`applyUrl` and a truncated `description`.

## 2. Read one requisition in full

```
GET /api/mcp/jobs?tool=get_job&jobId=14329
```

`jobId` is the `requisitionId` from a search result and is required — omitting it
returns `400 {"error":"jobId parameter is required"}`. The full record adds
`departmentId`, `address`, `city`, `state`, `zipCode`, `country`, `salaryMin`,
`salaryMax`, `salRateType`, `recruiterAssigned`, `brandId`, `createdAt` and
`updatedAt`, and the full HTML `description`. Note the envelope changes shape
here: the payload is a singular `result` object, not `results[]`.

## 3. Roll up by department or location

```
GET /api/mcp/jobs?tool=list_departments
GET /api/mcp/jobs?tool=list_locations
```

`list_locations` returns `results` as an **object** (`{states:[...], countries:[...]}`),
not an array. Parse the four tools separately; there is no single response shape.

## Gotchas that produce silent zeros

1. **Department filtering takes the numeric id, not the name.**
   `list_departments` returns `{"id":"27187","name":"27187"}` — the human label is
   never exposed there. `department=27187` works (167 of 229 jobs);
   `department=Manufacturing%20%26%20Operations` returns `totalCount: 0` with no
   error. To map an id to a label, call `get_job` on any posting and read
   `department` alongside `departmentId`.
2. **An unknown `tool` fails loudly, a bad filter value fails silently.**
   `tool=bogus` returns `400` and helpfully enumerates the valid set; a filter
   that matches nothing returns `200` with an empty `results[]`. Check
   `totalCount` before concluding there are no openings.

## Error handling

Failures are `application/json` with a flat `{"error": "<sentence>"}` body — not
RFC 9457, and with no machine-stable code, so match on HTTP status rather than on
the prose. Observed: `400` for an unknown tool or a missing required parameter,
`405` for any non-GET method. No `401`/`403` exists (there is no auth) and no
`429` was observed (no rate limit is signalled). See
`errors/graphic-packaging-problem-types.yml`.

## Rate limits and caching

None published and none signalled — no `RateLimit-*`, no `Retry-After`. Responses
carry `Cache-Control: public, max-age=60, s-maxage=120` and a strong `ETag`, so
poll conditionally and no faster than the 60-second cache. Treat the absence of a
documented quota as a reason to be conservative, not as permission.

## Applying

The API does not accept applications. `applyUrl` hands off to SAP SuccessFactors
(`career55.sapsf.eu?company=graphicpac&career_job_req_id=<requisitionId>`). This
surface is read-only end to end: there is no write operation, so nothing done
through it needs to be reversed.
