# Reliability verification map

This document records what the imported Vexa foundation already covers and what still requires a deployed staging run. It deliberately separates static/test evidence from live evidence.

## Duplicate-bot protection

The meeting API already exposes an atomic guarded-create seam for `(user, platform, native_meeting_id)`. The documented implementation combines:

- an active-row deduplication check,
- a per-user advisory-lock serialization boundary,
- a database unique partial-index backstop,
- a single-flight sweep guard for cross-replica scheduled work,
- durable lifecycle/session state,
- stale stopping/pre-active reconciliation,
- and explicit continuation semantics for terminal meetings.

The repository includes offline tests for single-flight contention, guarded concurrent spawning, retry idempotency, lifecycle durability, auto-join occurrence handling, stale-workload reconciliation and stress behavior. These tests are valuable regression evidence, but they do not prove that a deployed browser bot cannot duplicate itself under every infrastructure failure.

## Transcript reliability

The meetings domain documents and implements a separate collector path:

```text
bot capture → Redis transcription_segments stream → collector consumer group
→ durable transcript store → live transcript projection/API
```

The repository includes tests for collector health, pending-entry visibility, stream retention, transcript merging, database writing, recording chunks, stress floods, and delayed/restarted processing. Capture and transcription completion are intentionally separate states.

This addresses the failure mode where a recording exists while live transcript processing is delayed: the pending-entry/lag health surface and durable collector path provide an observable recovery point. A deployed environment still needs an operator reconciliation policy for a recording whose stream has been permanently lost.

## Required staging evidence

Before calling Open BOT production-ready, run the Compose or Kubernetes stack with a test account and capture evidence for:

1. Two concurrent `POST /bots` requests for the same meeting: one active bot only.
2. Repeated calendar sweep ticks during a 60-minute meeting: no second bot.
3. Meeting API restart while the bot remains active: lifecycle rehydration and no duplicate spawn.
4. Long meeting with continuous transcript segments: ordered, complete transcript.
5. Large meeting with temporary transcription-worker interruption: recording retained, collector health degraded visibly, segments recovered or reconciliation reported.
6. Consumer restart with Redis pending entries: pending entries reclaimed and persisted once.
7. Meeting completion: bot, transcript and recording reach independently queryable terminal state.

Record deployment revision, service logs, bot status, transcript segment counts, recording metadata, and failure/recovery timestamps. Do not store real customer recordings or transcripts in Git.

## Current conclusion

Static source review found substantial existing lifecycle and collector protection; it did not justify a speculative production-code change. The next code changes should be driven by a failed staging case or a reproducible failing test. The absence of live infrastructure in this review is a verification gap, not evidence that the system is fully validated.
