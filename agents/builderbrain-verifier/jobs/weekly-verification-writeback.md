# Weekly Verification And Writeback

## Objective

Execute a weekly verification sweep for Builder Brain system health and write results back to Builder Brain as durable progress.

## Steps

1. Load canonical state:
   - Use `BASE_URL="${BUILDER_BRAIN_BASE_URL:-${AGENT_BUILDER_BRAIN_BASE_URL:-http://172.17.0.1:8001}}"` (fallback `http://127.0.0.1:8001`).
   - Resolve the shared secret in this order:
     - `SECRET="${BUILDER_BRAIN_SHARED_SECRET:-${AGENT_BUILDER_BRAIN_SHARED_SECRET:-${BUILDER_BRAIN:-}}}"`
     - If `SECRET` is still empty and `skills/library/agent-job-secrets/agent-job-secrets.js` exists, run `node skills/library/agent-job-secrets/agent-job-secrets.js get BUILDER_BRAIN` and use its stdout as `SECRET`.
   - If `SECRET` is still empty, stop and report: `Builder Brain secret is not configured in environment`.
   - `curl -fsS -H "x-openclaw-shared-secret: ...secret..." "$BASE_URL/api/projects/builder_brain_system_source_of_truth"`
   - `curl -fsS -X POST "$BASE_URL/api/tools/retrieve_context" -H "Content-Type: application/json" -H "x-openclaw-shared-secret: ...secret..." -d '{"message":"weekly verification targets and known risks for builder_brain_system_source_of_truth","user_id":"popebot","session_id":"weekly-verifier","project_key":"builder_brain_system_source_of_truth","answer":true}'`
2. Verify live endpoints:
   - `curl -fsS "$BASE_URL/health"`
   - `curl -I https://openclaw.infinitebarrz.cloud`
   - `curl -I https://cb.infinitebarrz.cloud`
3. Verify Qdrant network path from this host:
   - `curl -fsS --max-time 10 http://172.17.0.1:32768/collections`
   - If unreachable, capture exact error output.
4. Build a concise findings summary:
   - What passed
   - What failed
   - Immediate next actions
5. Save verified results:
   - `curl -fsS -X POST "$BASE_URL/api/tools/save_project_progress" -H "Content-Type: application/json" -H "x-openclaw-shared-secret: ...secret..." -d '{"project_key":"builder_brain_system_source_of_truth","summary":"<summary>","detail":"<detail>","status":"completed","phase":"weekly-verification","user_id":"popebot","metadata":{"saved_via":"popebot_weekly_verifier"}}'`
   - Success requires non-empty JSON `id`.
6. If major status changed, also save a handoff:
   - `curl -fsS -X POST "$BASE_URL/api/projects/handoff" -H "Content-Type: application/json" -H "x-openclaw-shared-secret: ...secret..." -d '{"project_key":"builder_brain_system_source_of_truth","title":"<title>","summary":"<summary>","current_status":"<current_status>","current_objective":"<current_objective>","next_steps":<next_steps_json>,"priority":0,"updated_at":"<ISO8601_UTC>","source":"popebot_weekly_verification","tags":<tags_json>,"metadata":{}}'`
   - Success requires non-empty JSON `project_key`.

## Constraints

- Do not mark checks as successful without command evidence.
- Do not overwrite architecture doctrine.
- Keep writeback factual and specific.
