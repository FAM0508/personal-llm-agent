# Personal LLM Agent

> A self-hosted, always-on LLM agent with long-term memory, tool use, and autonomous nightly reflection. Built on FastAPI, DeepSeek, Claude, and sqlite-vec. Runs on a single 2GB VPS.

## Overview

This project explores what happens when you treat an LLM agent as a **long-running process** with its own filesystem, its own memory, and its own circadian rhythm — rather than as a stateless API call.

- **Physical footprint**: 1 uvicorn process + 7 cron jobs on a 2GB VPS
- **Interaction**: Messaging app webhook (single user, single tenant)
- **Persistence**: Flat files + sqlite-vec + FTS5 (no containers, no ORM)
- **Runtime cost**: ~$2K/month LLM API budget (DeepSeek primary, Claude for heavy tasks)

## Architecture

### Four-Layer Memory

1. **STM (Short-term)** — rolling conversation window, auto-compressed above 30 turns
2. **Episodic** — append-only JSONL, all conversations persisted at `chmod 600`
3. **LTM (Long-term)** — `sqlite-vec` (1024-dim vectors) + FTS5 (BM25) + audit log
4. **Persona** — 7 Markdown files defining identity, values, and curated memories

### Three-Way Hybrid Retrieval

Every user query triggers three parallel recall paths:

- **Vector recall** (cosine KNN top-16) via `bge-m3` embeddings
- **FTS5 full-text recall** with BM25 ranking and input sanitization
- **Recency recall** from the last N episodes

Merged by `memory_id`, scored by `0.5·similarity + 0.3·importance + 0.2·recency + 0.05·multi-path-hit`, then rate-limited per source (FTS ≥ 2, Recent ≥ 2) to prevent single-path dominance.

Relevance filtering is delegated to an LLM guardrail rather than a fixed similarity threshold — a deliberate architectural choice after observing threshold drift on small corpora.

### Autonomous Nightly Cycle
01:00 suggestions cleanup — purge stale unconfirmed memories
02:00 digestion — 10-organ filesystem housekeeping
02:50 digestion circuit-break — yield to dream
03:00 dream — read daily delta → LLM distillation → write SUGGESTIONS.md
04:00 dream retry 1
05:00 dream retry 2 + offsite backup
08:55 self-check — 14-item health probe
09:00 morning report
09:05 dream report — push last night's suggestions
09:30 deadman switch — heartbeat watchdog

### Safety Layers

**Tool-call sandbox (`safe_bash`)** — 6 defense layers:
1. Metacharacter rejection (`|`, `>`, `<`, `&`, `;`, `$`, backtick, `*`, `?`, `{}`, `[]`, `~`)
2. First-token whitelist (23 probes)
3. Recursive flag rejection (`-R`, `-r`, `-f`, `--recursive`)
4. Path hard-deny (`/etc/`, `.env`)
5. Device-file rejection (`/dev/sd*`, `/proc/kcore`)
6. Command length ≤ 512 chars, clean env, 15s timeout, 2000-char output truncation

**Identity lock** — Six persona files (`IDENTITY`, `SOUL`, `KEYSTONE`, `USER`, `AGENT`, `MEMORY`) are protected by process-level constants + filesystem `chmod 444` + git pre-commit hook. The `dream` process can only write to `SUGGESTIONS.md` and `reflect_signals.jsonl`.

**Conversation lock (V7)** — per-sender `fcntl` file lock with 60s circuit breaker; prevents context cross-contamination from concurrent messages.

**Message idempotency** — 500-entry sliding window keyed by `message_id`, protects against webhook retries.

## Key Engineering Decisions

- **Flat files over databases** for persona — human-readable, git-friendly, diff-able, reviewable
- **Append-only memory with 30-day audit rollback** — no destructive writes on long-term store
- **LLM guardrail > similarity threshold** — thresholds drift; guardrails scale
- **Separate `root` and `aime` cron domains** — filesystem ownership isolation prevents privilege cascading

## Reliability Patterns

- **Deadman switch**: self-check writes heartbeat; separate cron validates staleness
- **Four-blade dream hardening**: webhook heartbeat touch, reflect-signal injection, NULL-guard cursor, poison-batch DLQ
- **NULL guard**: prevents cursor over-advance when LLM returns degenerate output ("nothing to distill")
- **Message degradation chain**: Markdown card → plain card → chunked text → plain text (never drops a message)

## Known Limitations

Documented openly in a `REALITY_PATCH` file committed to the repo — a deliberate design to prevent the agent from developing inaccurate self-models:

- The routing brain is **flat fallback**, not a multi-tier architecture
- The self-evolution module is a **stub**, not a live feature
- Only the components listed above are in production; roadmap modules are in sandbox only

Honesty over marketing — especially when the product is an agent that reads its own documentation.

## Status

Single-tenant, personal use. Stable operation ≥ 11 days, MTBF ≥ 72h, p99 response < 8s.

Not open-sourced. This README is for portfolio reference.

---

Built with DeepSeek, Claude, sqlite-vec, FTS5, FastAPI, systemd, cron, Git.
