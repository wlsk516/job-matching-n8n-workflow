# AI Job Matching Workflow (n8n)

This is an n8n workflow that takes my fixed client profile (resume + expected salary) and matches it against job listings from the OpenWebNinja JSearch API.

## What it does

- Fetches job listings from OpenWebNinja JSearch API (multiple HTTP Request nodes).
- Cleans and merges responses, removes duplicates, and flattens the data.
- Uses an AI Agent (OpenAI Chat Model) to:
  - Score each job (0–10) for fit against the profile.
  - Enforce salary rules (only pass if salary is above a floor or unknown).
  - Return structured JSON with `match_score`, `match_reason`, and key job details.
- Filters to jobs with `match_score >= 6` and aggregates them into a summary message.

## Files

- `job-matching-workflow.json` – export of the n8n workflow.

## How to run

1. Import `job-matching-workflow.json` into your n8n instance.
2. Set environment variables / credentials:
   - `OPENWEBNINJA_API_KEY`
   - `OPENAI_API_KEY`
3. Activate the workflow and execute.

## Common Issues

### Gateway timeout from JSearch API

- **Symptom:** Some HTTP Request nodes to the OpenWebNinja JSearch API would fail with a 504-style gateway timeout when requesting many pages or broad queries.
- **Cause:** The upstream API occasionally timed out on heavier queries, which caused the whole workflow execution to fail.
- **How I handled it:**
  - Reduced the number of pages per request to keep responses smaller and faster.
  - Added basic retry behavior in n8n for transient failures.
  - Logged failing requests to quickly see which queries were unstable.

### "No data array" / empty results

- **Symptom:** Downstream nodes (Item Lists / AI Agent) would fail because they expected a `data` array, but the API sometimes returned an empty or differently shaped payload.
- **Cause:** For some queries, the JSearch API returned no matches or a slightly different JSON structure, so the `data` array was missing or empty.
- **How I handled it:**
  - Added a check in n8n to verify that `data` exists and `data.length > 0` before mapping items.
  - If there is no data, the workflow now short-circuits and returns a friendly "No matching jobs found" message instead of throwing an error.
  - Normalized the JSON structure with helper nodes so downstream nodes always see a consistent `data` array.
