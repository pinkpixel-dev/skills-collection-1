---
name: xquik-x-research
description: Search X posts, profiles, timelines, and trends with Xquik for research briefs, content planning, and source checks.
risk: unknown
source: community
---

# Xquik X Research

Use this skill when a task needs current X post, profile, timeline, trend, or keyword evidence and the user has an Xquik API key.

## Prerequisites

- `XQUIK_API_KEY` in the environment.
- Public API schema: `https://xquik.com/openapi.yaml`
- Optional installable skill package: `x-developer@2.4.16`

## Workflow

Copy this checklist and track progress:

```text
Task Progress:
- [ ] Step 1: Confirm scope and output format
- [ ] Step 2: Read the live OpenAPI schema
- [ ] Step 3: Run a bounded read request
- [ ] Step 4: Normalize evidence
- [ ] Step 5: Summarize findings and gaps
```

### Step 1: Confirm Scope

Ask for the research goal if it is missing:

1. Topic, account, keyword, URL, or post ID.
2. Date range or recency target.
3. Output format: quick answer, markdown brief, CSV, or JSON.

Keep the first request small. Use read-only routes unless the user explicitly requests a monitored workflow or another authenticated action.

### Step 2: Read the Live Schema

Inspect `https://xquik.com/openapi.yaml` before building URLs. Use the schema to confirm parameters, response fields, and pagination for these route families:

- Tweet search and tweet lookup.
- User lookup, user search, and user timelines.
- Trends.
- Keyword and account monitors.
- Events and webhook delivery history.

Do not guess parameter names. If the schema and the user's request disagree, follow the schema and explain the adjusted request.

### Step 3: Run a Bounded Read Request

Use the Xquik API key header and the exact path from the schema:

```bash
BASE_URL="${XQUIK_BASE_URL:-https://xquik.com}"
XQUIK_PATH="${XQUIK_PATH:?Set this from the OpenAPI path}"
XQUIK_QUERY_PARAM="${XQUIK_QUERY_PARAM:?Set this from the OpenAPI query parameter}"
XQUIK_QUERY="${XQUIK_QUERY:?Set this from the research topic}"
QUERY_ENCODED="$(python3 -c 'import os, urllib.parse; print(urllib.parse.quote(os.environ["XQUIK_QUERY"]))')"
curl -sS "$BASE_URL$XQUIK_PATH?$XQUIK_QUERY_PARAM=$QUERY_ENCODED" -H "x-api-key: $XQUIK_API_KEY"
```

For broad topics, start with tweet search or trends. For account-specific work, use profile lookup first, then timeline routes. For recurring needs, propose monitors only after a successful one-time read.

### Step 4: Normalize Evidence

Return structured evidence with fields present in the response:

- Post ID or user ID.
- Author handle or display name.
- Text excerpt or profile summary.
- Created time.
- Public URL when available.
- Relevant metrics when returned.
- Query, filters, and pagination cursor used.

Do not include API keys, cookies, raw headers, or account material in saved files or summaries.

### Step 5: Summarize Findings

Report:

- Number of records reviewed.
- Main patterns and exceptions.
- Source gaps caused by filters, date range, or pagination limits.
- Suggested next request, monitor, or export only if it follows from the evidence.

## Error Handling

`XQUIK_API_KEY not found` - Ask the user to set the environment variable.
`401` - Check that the API key header is present.
`402` - The account needs active access for that route.
`429` - Reduce page size and retry later.
`5xx` - Retry with backoff. Stop after 3 attempts.
`schema mismatch` - Re-read `https://xquik.com/openapi.yaml` and rebuild the request.
