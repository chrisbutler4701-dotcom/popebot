# Weekly Verification And Writeback

## Objective

Execute a weekly verification sweep for Builder Brain system health and write results back to Builder Brain as durable progress.

## Steps

1. Load canonical state:
   - `builder-brain get-project builder_brain_system_source_of_truth`
   - `builder-brain retrieve-context "weekly verification targets and known risks for builder_brain_system_source_of_truth"`
2. Verify live endpoints:
   - `curl -fsS http://127.0.0.1:8001/health`
   - `curl -I https://openclaw.infinitebarrz.cloud`
   - `curl -I https://cb.infinitebarrz.cloud`
3. Verify Qdrant network path from this host:
   - `curl -fsS --max-time 10 http://127.0.0.1:32768/collections`
   - If unreachable, capture exact error output.
4. Build a concise findings summary:
   - What passed
   - What failed
   - Immediate next actions
5. Save verified results:
   - `builder-brain save-progress builder_brain_system_source_of_truth "<summary>" "<detail>" "completed" "weekly-verification" "popebot"`
6. If major status changed, also save a handoff:
   - `builder-brain save-handoff builder_brain_system_source_of_truth "<title>" "<summary>" "<current_status>" "<current_objective>" 0 "popebot_weekly_verification" "<next_steps_json>" "<tags_json>" "{}"`

## Constraints

- Do not mark checks as successful without command evidence.
- Do not overwrite architecture doctrine.
- Keep writeback factual and specific.

