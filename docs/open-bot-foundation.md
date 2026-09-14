# Open BOT foundation audit

This repository is the self-hosted meeting capture foundation for Open BOT. It is based on the upstream Vexa project and retains the upstream Apache-2.0 license and attribution.

## Current verified foundation

The current tree includes:

- Meeting capture services under `core/meetings`
- Gateway and API contracts under `core/gateway`
- Runtime and workload orchestration under `core/runtime`
- Docker Compose deployment under `deploy/compose`
- Lite deployment under `deploy/lite`
- Kubernetes/Helm deployment under `deploy/helm`
- Separate transcription deployment under `deploy/transcription`
- Workspace packages and lockfiles managed with pnpm and Turbo
- Repository validation gates under `scripts/`

The upstream README describes support for Google Meet, Microsoft Teams and Zoom, real-time transcripts, speaker attribution, recordings, Docker deployment and Kubernetes deployment. These capabilities must be validated in our own staging environment before being marketed as production-ready for Open BOT.

## Open BOT product boundary

Open BOT-specific product logic should be added around the capture foundation rather than mixed into upstream services unnecessarily. The integration boundary should cover:

1. Calendar/provider event ingestion.
2. Idempotent bot dispatch keyed by platform and native meeting ID.
3. Bot status, lease and heartbeat reconciliation.
4. Durable transcript and recording ingestion.
5. Post-meeting processing and downstream integrations.

The original MeetOS repository remains separate and is not modified by this project.

## Reliability requirements

### Duplicate bot prevention

A meeting must have at most one active bot for a `(platform, native_meeting_id)` key. The integration must use a durable uniqueness rule and reconcile desired state with observed bot/runtime state after retries, process restarts and worker failures. An in-memory set is not sufficient.

Required tests:

- Repeated dispatch requests for the same meeting.
- Concurrent dispatch requests.
- Controller restart while a bot is active.
- Stale heartbeat and worker recovery.
- Recurring calendar events with distinct occurrence IDs.
- A meeting longer than 40 minutes.

### Large-meeting transcription

Capture completion must not be treated as transcription completion. Transcript segments need incremental durable persistence, backpressure/queue visibility, retries and a reconciliation path for recordings whose live transcript stream is incomplete.

Required tests:

- Large participant count.
- Long meeting with continuous transcript segments.
- Temporary transcription-worker outage.
- Consumer restart during an active meeting.
- Recording present but transcript delayed or incomplete.
- Final transcript completeness and ordering checks.

## Delivery sequence

1. Establish a reproducible local/CI validation path for the imported foundation.
2. Map the meeting API, bot manager, runtime, transcript collector and storage contracts.
3. Add the idempotent bot lifecycle adapter.
4. Add transcript durability and reconciliation.
5. Add calendar integration independently from the capture engine.
6. Connect the existing MeetOS summary/action-item workflow through stable APIs.
7. Run short, long and large-meeting staging tests.
8. Document deployment profiles, data residency controls, retention and deletion.

## Security and data handling

Never commit `.env` files, OAuth refresh tokens, API keys, browser sessions, recordings, real transcripts or database dumps. Deployments must supply secrets through the target platform's secret manager. Regional deployment and data-residency controls are technical capabilities, not a blanket legal-compliance claim.

## Validation status

This document records the initial source audit. No live meeting, production deployment or real customer data was used for this audit. The next change must include executable validation and report any checks that require Docker, GPU, credentials or staging infrastructure.
