---
title: Persist cloned voices across service restarts
date: 2026-11-11
summary: "Made API-enrolled VieNeu voices durable, rebuilt the service, and verified three cloned voices after restart."
---

# Persist cloned voices across service restarts

## What happened

`POST /v1/voices` previously mutated only the running model. The API now uses `apps.user_voices` to atomically persist clones under `VIENEU_HOME`, and startup reloads them before warm-up. Enrollment and deletion share one process lock so concurrent writers cannot replace a newer snapshot.

Root `compose.yml` now mounts a dedicated `vieneu_voices` volume at `/home/app/.vieneu`. Both production Dockerfiles create the cache and voice directories with `app` ownership before switching users.

The three David reference clips were edge-trimmed with `V3TurboVieNeuTTS._preclean_reference_audio`, enrolled, and exercised after a full service restart.

## Decision

Capture built-in presets and aliases before loading user voices. This keeps built-in replacement protection while allowing a persisted user voice to be re-enrolled. Keep voice data separate from the disposable Hugging Face cache.

## Verification

- `uv run pytest tests/test_user_voices.py tests/test_openai_speech.py -q`: 23 passed.
- `podman-compose config`: resolved `VIENEU_HOME` and both named volumes.
- All trimmed references were readable mono WAVs and shorter than their sources.
- All three enrollments returned HTTP 200; the JSON store was non-empty and writable by UID 1000.
- After restart, each ID appeared exactly once and synthesized a valid non-empty WAV.

## Next steps

None.

> Historical work record — not durable authority. Prefer docs/specs/ADRs for current decisions.
