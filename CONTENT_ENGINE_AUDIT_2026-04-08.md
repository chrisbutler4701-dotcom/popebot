# Content Engine Throughput And Publish-Path Audit

Date: 2026-04-08
Repository: `movement.barrzflow`
Decision: No PR opened. I did not find a fix that could be fully verified from this environment.

## Scope

This audit traced the only enabled scheduled flow in the repository, validated reachable runtime dependencies where possible, and checked whether the configured workflow can prove end-to-end publish integrity.

## Executive Summary

The repository does not currently contain a multi-stage content engine. The only enabled automation is a weekly Builder Brain verification/writeback agent wired through [`agent-job/CRONS.json`](agent-job/CRONS.json) and [`agents/builderbrain-verifier/jobs/weekly-verification-writeback.md`](agents/builderbrain-verifier/jobs/weekly-verification-writeback.md). Public endpoints are reachable, but the local dependencies required by the verifier were not reachable from this environment, and the Builder Brain secret/base URL variables were unset at runtime. As configured, throughput is limited to a single weekly agent run, and publish-path integrity depends on prompt-following rather than an executable, self-verifying pipeline.

## Validated Flow

1. Scheduler launches `builderbrain-weekly-verification-writeback` once per week on Mondays at 13:00 UTC.
2. That job reads the Builder Brain verifier system prompt and the weekly job prompt.
3. The job is expected to:
   - read canonical Builder Brain state,
   - hit health endpoints,
   - probe Qdrant,
   - summarize findings,
   - write progress and optionally a handoff back into Builder Brain.

Evidence:
- [`agent-job/CRONS.json:17`](agent-job/CRONS.json#L17) enables the only active job and schedules it weekly.
- [`agents/builderbrain-verifier/SYSTEM.md:7`](agents/builderbrain-verifier/SYSTEM.md#L7) defines the mission as weekly grounded verification plus durable writeback.
- [`agents/builderbrain-verifier/jobs/weekly-verification-writeback.md:9`](agents/builderbrain-verifier/jobs/weekly-verification-writeback.md#L9) through [`agents/builderbrain-verifier/jobs/weekly-verification-writeback.md:31`](agents/builderbrain-verifier/jobs/weekly-verification-writeback.md#L31) define the read, verify, and save steps.

## Findings

### 1. Critical: End-to-end publish path is not runnable from the audited environment

The verifier requires Builder Brain connectivity and a shared secret before it can even read canonical state. At runtime, `BUILDER_BRAIN_BASE_URL`, `AGENT_BUILDER_BRAIN_BASE_URL`, `BUILDER_BRAIN_SHARED_SECRET`, and `AGENT_BUILDER_BRAIN_SHARED_SECRET` were all unset. The job prompt explicitly says to stop when the secret is missing, so the publish path cannot complete here.

Evidence:
- [`agents/builderbrain-verifier/jobs/weekly-verification-writeback.md:10`](agents/builderbrain-verifier/jobs/weekly-verification-writeback.md#L10) through [`agents/builderbrain-verifier/jobs/weekly-verification-writeback.md:13`](agents/builderbrain-verifier/jobs/weekly-verification-writeback.md#L13) require base URL and secret configuration and instruct the agent to stop if the secret is empty.
- Runtime check result:

```text
BUILDER_BRAIN_BASE_URL=
AGENT_BUILDER_BRAIN_BASE_URL=
BUILDER_BRAIN_SHARED_SECRET=
AGENT_BUILDER_BRAIN_SHARED_SECRET=
```

Impact:
- No verified read from Builder Brain.
- No verified save via `save_project_progress`.
- No verified handoff via `projects/handoff`.

### 2. High: Local dependency checks fail, so integrity coverage is incomplete

The weekly verifier expects local access to the event handler and Qdrant-adjacent network path, but both checks failed from this workspace.

Evidence:
- [`docker-compose.override.yml:3`](docker-compose.override.yml#L3) maps the event handler to `127.0.0.1:8088`.
- [`agents/builderbrain-verifier/jobs/weekly-verification-writeback.md:16`](agents/builderbrain-verifier/jobs/weekly-verification-writeback.md#L16) and [`agents/builderbrain-verifier/jobs/weekly-verification-writeback.md:20`](agents/builderbrain-verifier/jobs/weekly-verification-writeback.md#L20) require local health and Qdrant checks.
- Runtime check results:

```text
$ curl -fsS -D - http://127.0.0.1:8088/api/ping -o /dev/null
curl: (7) Failed to connect to 127.0.0.1 port 8088 after 2 ms: Couldn't connect to server

$ curl -fsS --max-time 10 -D - http://172.17.0.1:32768/collections -o /dev/null
curl: (7) Failed to connect to 172.17.0.1 port 32768 after 3 ms: Couldn't connect to server
```

Impact:
- The current verifier cannot establish whether the internal publish path is healthy.
- Any saved status would at best describe partial availability, not full path integrity.

### 3. High: Throughput is capped at one weekly run with no retry lane

Only one cron job is enabled in the whole repo, and it runs weekly. There is no active event-driven trigger, no retry job, and no higher-frequency publish or verification path.

Evidence:
- [`agent-job/CRONS.json:17`](agent-job/CRONS.json#L17) through [`agent-job/CRONS.json:21`](agent-job/CRONS.json#L21) show a single enabled job.
- [`event-handler/TRIGGERS.json:3`](event-handler/TRIGGERS.json#L3) through [`event-handler/TRIGGERS.json:56`](event-handler/TRIGGERS.json#L56) show every webhook/GitHub trigger disabled.

Impact:
- Best-case throughput is one verification/writeback cycle per week.
- Any transient failure creates at least a week of blind time unless someone intervenes manually.

### 4. Medium: “Publish integrity” is prompt-defined, not mechanically enforced

The job prompt describes success criteria for save operations, but those checks are not implemented as code in the repo. Integrity therefore depends on the agent following instructions rather than a reusable script or testable command wrapper.

Evidence:
- [`agents/builderbrain-verifier/SYSTEM.md:20`](agents/builderbrain-verifier/SYSTEM.md#L20) through [`agents/builderbrain-verifier/SYSTEM.md:23`](agents/builderbrain-verifier/SYSTEM.md#L23) define behavioral rules only.
- [`agents/builderbrain-verifier/jobs/weekly-verification-writeback.md:26`](agents/builderbrain-verifier/jobs/weekly-verification-writeback.md#L26) through [`agents/builderbrain-verifier/jobs/weekly-verification-writeback.md:31`](agents/builderbrain-verifier/jobs/weekly-verification-writeback.md#L31) specify success conditions in prose.

Impact:
- There is no repo-local executable that can be run in CI or locally to prove save-path behavior.
- Regressions in payload shape, authentication, or response validation would be caught only during a live agent run.

### 5. Low: Public ingress is up, but that does not prove the internal publish chain

The two public domains named by the verifier returned HTTP 200 during the audit.

Evidence:

```text
$ curl -fsSI --max-time 15 https://openclaw.infinitebarrz.cloud
HTTP/2 200

$ curl -fsSI --max-time 15 https://cb.infinitebarrz.cloud
HTTP/2 200
```

Impact:
- External ingress is available.
- This does not validate Builder Brain read/write auth, local event-handler availability, or Qdrant reachability.

## Bottlenecks

1. Secret-gated start condition with no configured secret in the audited environment.
2. Single weekly schedule with no retry/backfill mechanism.
3. No enabled event-driven triggers to increase throughput or shorten feedback time.
4. No executable verifier script, so the path is hard to test outside a live agent run.
5. Internal dependencies are coupled to fixed host/port assumptions (`127.0.0.1:8088`, `172.17.0.1:8001`, `172.17.0.1:32768`).

## Recommended Next Implementation

This is the lowest-risk implementation sequence that would make the flow testable and materially improve integrity:

1. Extract the verifier logic into a repo-local shell script, for example `agents/builderbrain-verifier/bin/weekly_verification_writeback.sh`.
   - Fail hard on missing secret/base URL.
   - Emit machine-checkable pass/fail for each stage.
   - Validate that save responses contain non-empty `id` and `project_key`.
2. Add a dry-run mode to the script.
   - Allow endpoint and payload validation without mutating Builder Brain.
3. Add a higher-frequency verification cron.
   - Example: hourly dry-run health verification plus weekly mutating writeback.
4. Convert at least one trigger path from disabled to active once the script exists.
   - Prefer a narrow retry/health trigger rather than a broad webhook auto-publish path.
5. Document required environment variables in `.env.example`.
   - Add Builder Brain URL/secret variables explicitly so deployment drift is visible.

## Verification Needed Before Any PR

Do not open a fix PR for this path until all of the following are available in one environment:

1. Builder Brain URL and shared secret.
2. Reachable local event-handler health endpoint.
3. Reachable Qdrant endpoint or an updated canonical endpoint for vector storage.
4. A dry-run or mocked save-path test proving request/response handling.
5. At least one successful end-to-end run with captured evidence for:
   - canonical state read,
   - endpoint checks,
   - save response with non-empty `id`,
   - optional handoff response with non-empty `project_key`.

## Why No PR Was Opened

I did not make runtime changes because this environment cannot fully verify the publish path. Any fix to the agent prompt, schedule, or writeback behavior would still leave the critical uncertainty unresolved: whether Builder Brain and the internal network dependencies are correctly provisioned and reachable in the target runtime.
