# TurboScript — token economics analysis

Analysis of OpenCode v2 (`dev`, 2.3.0) at the time of the fork, plus real usage data from
`~/.local/share/opencode/opencode.db`. The goal is a coding-focused build that spends fewer
tokens per completed task without losing capability.

## 1. Method

- Read the request-building, history, compaction, and tool-output code paths in `packages/core`
  and `packages/ai`.
- Aggregated every assistant message and tool part in the local database (1,297 assistant
  messages across all sessions).

## 2. Measured baseline

| Metric | Value |
| --- | --- |
| Assistant responses | 1,297 |
| Fresh input tokens | 3,491,652 |
| Cache-read tokens | 61,016,448 |
| **Prompt tokens processed** | **64,508,100** |
| Output tokens | 564,307 |
| Average prompt per request | ~49,800 |
| Cache writes reported | 0 (provider does not report them) |

Largest sessions by prompt tokens:

| Session | Messages | Input | Cache read |
| --- | --- | --- | --- |
| Install Google Chrome on Linux | 236 | 648k | 14.1M |
| WhatsApp automation with wa.mjs | 143 | 139k | 9.3M |
| Callback URL explanation | 116 | 487k | 8.3M |
| Diagnosing Instagram force quit on AVD | 80 | 491k | 6.2M |

A 236-message session processed 14.8M prompt tokens. That is the replayed conversation being
re-read on almost every step.

### Tool results dominate the payload

Model-visible tool output, by call count and characters:

| Tool | Calls | Characters |
| --- | --- | --- |
| `read` | 257 | 4,194,683 |
| `bash` | 1,016 | 3,646,469 |
| `edit` | 131 | 352,667 |
| `write` | 47 | 151,144 |
| `websearch` | 9 | 128,665 |
| `grep` | 48 | 98,393 |
| `todowrite` | 21 | 63,170 |
| `webfetch` | 4 | 58,054 |

Distribution of `read` + `bash` outputs (model-visible text only):

| Size bucket | Calls | Characters |
| --- | --- | --- |
| < 500 | 635 | 102,760 |
| 500 – 2k | 325 | 359,403 |
| 2k – 8k | 233 | 874,497 |
| 8k – 20k | 53 | 645,423 |
| 20k – 50k | 20 | 591,907 |
| > 50k | 7 | 110,104 |

7% of tool calls (80) produce half of all tool-result characters. Those results are then
replayed in every subsequent request until compaction. This is the single largest lever.

## 3. What already works (do not rebuild)

- **Prompt caching** with automatic breakpoints (`packages/ai/src/cache-policy.ts`): tools,
  first/last system parts, conversation tail, 4-breakpoint budget, per-provider TTL.
- **Auto compaction** with background precompute, checkpoint template, cache-aware lead time
  (`packages/core/src/session/compaction.ts`): `keep` 15k tokens, 15% / minimum 16k buffer.
- **Tool-output truncation** at 2,000 lines / 50 KiB, spilling the full text to a file and
  printing the path (`packages/core/src/tool-output.ts`).
- **Image pruning** over a byte budget, in stable batches to preserve the cache
  (`boundImages` in `packages/core/src/session/model-request.ts`).
- **Instruction diffs** instead of re-sending changed `AGENTS.md` files
  (`packages/core/src/instruction-discovery.ts`).
- **Small-model title generation** with a minimal variant
  (`packages/core/src/session/context.ts`, `selectTitle`).
- **Output limits fitted to remaining window** (`outputLimit`).
- **Step-limit cache preservation**: tool definitions stay on the last step even when tools
  are disabled (`packages/core/src/session/runner/llm.ts:279`).

## 4. Gaps, ranked by expected savings

### 4.1 Age out old tool results (largest win)

Nothing prunes completed tool results. `toLLMMessages` replays every result verbatim for the
whole session (`packages/core/src/session/runner/to-llm-message.ts`). A single 30k-character
`read` is paid again on every later step.

Proposal: keep the last N tool results (N ≈ 6) verbatim; replace older ones with a one-line
stub carrying tool name, target, size, and spill-file path. Applied in stable generations
(only when the boundary crosses a step multiple) so the cached prefix stays stable between
cuts.

Expected: 30–50% fewer prompt tokens processed in long sessions; little effect on short ones.

### 4.2 Tighter per-call output caps for coding work

Current caps: `read` 2,000 lines / 50 KiB, tool output 2,000 lines / 50 KiB
(`packages/core/src/tool/read-filesystem.ts:13`, `packages/core/src/tool-output.ts:11`).
Measured 80 oversized calls account for 1.35M characters.

Proposal: default ~400 lines / 12 KiB with the spill path retained, plus an outline/skeleton
mode in `read` so an agent can inspect a large file cheaply before paging into it.

Expected: 10–20% on sessions with large reads; also lowers first-step latency.

### 4.3 Unchanged-file read dedup

Re-reading a file that has not changed since the last read re-sends the full content.
Proposal: detect identical `(path, mtime/content hash, offset, limit)` and return a short
"unchanged since step N" stub instead.

Expected: 3–8%, concentrated in long edit loops.

### 4.4 Coding-only scope profile

Every built-in tool's schema is sent on every request. Analysis/creative surface
(`websearch`, `webfetch`, `image-generate`, `automation`, `artifact`, `question`, `skill`)
costs schema tokens and invites non-coding turns. MCP servers and subagents add more.

Proposal: a default profile that enables `read`, `edit`, `write`, `patch`, `grep`, `glob`,
`shell`, `todo`; no MCP by default; no subagents unless enabled; `steps` capped (currently
unlimited by default, `packages/schema/src/agent.ts`). Users opt back in per project.

Expected: 5–15% of static prefix, plus avoided analysis detours.

### 4.5 Cheaper auxiliary model routing

Title generation already prefers a small model. Compaction only does so when a `compaction`
agent model is configured; otherwise it uses the session model
(`packages/core/src/session/compaction.ts:275`). Compaction prompts include long transcripts
and up to 32k output tokens.

Proposal: default compaction (and generate) to the provider's small model when one exists,
with a config override.

Expected: 2–6% of total spend, more on compaction-heavy sessions.

### 4.6 Context bloat from instruction files and code mode

`AGENTS.md` is re-sent in full (this repo's is ~4k tokens). Code-mode instructions are 7.3 KB
(`packages/core/src/codemode/instructions.ts`) and only matter when code mode is on.

Proposal: byte budget for discovered instructions (`instructions.max_bytes`), and ensure
code-mode guidance is omitted when the flag is off.

Expected: 2–5% on repos with large instruction files.

### 4.7 Budget guard

There is no per-session token or cost ceiling. Proposal: `turbo.budget` with soft threshold
(auto-compact) and hard threshold (stop and report). Prevents runaway replays.

### 4.8 Output verbosity

564k output tokens over 1,297 responses (~435 each) is reasonable. The system prompt already
asks for concision; no change recommended beyond keeping tool-result stubs terse.

## 5. Proposed TurboScript shape

- Rebranded CLI/TUI (`turbo`), config `turbo.json(c)`, env prefix `TURBO_`, data dir
  `~/.local/share/turboscript`, server identity `turboscript`.
- "Turbo mode" on by default:
  - `turbo.context`: tool-result aging (4.1), tighter caps (4.2), read dedup (4.3).
  - `turbo.scope`: coding-only tool set and step cap (4.4).
  - `turbo.model`: auxiliary small-model routing (4.5).
  - `turbo.budget`: session ceilings (4.7).
- Every optimization individually switchable and reported in `turbo stats` (tokens processed,
  tokens saved by pruning, cache hit share).
- Benchmark harness: replay a recorded session's tool payloads and compare prompt tokens per
  completed task before/after.

## 6. Effort and risk

| Item | Effort | Risk | Savings estimate |
| --- | --- | --- | --- |
| 4.1 Tool-result aging | Medium | Cache churn if cut too often | 30–50% |
| 4.2 Tighter caps + outline | Low | Agents paging more | 10–20% |
| 4.3 Read dedup | Low | Stale reads if mtime unreliable | 3–8% |
| 4.4 Coding-only scope | Low | Missing tools for edge tasks | 5–15% + detour avoidance |
| 4.5 Aux model routing | Low | Weak summaries on small models | 2–6% |
| 4.6 Instruction budget | Low | Lost conventions | 2–5% |
| 4.7 Budget guard | Low | None (opt-in stop) | Prevents blowups |

Savings overlap; combined realistic target is 40–60% fewer prompt tokens per long coding
session, driven mostly by 4.1 and 4.2.

## 7. Decisions needed

1. Which items to implement in the first pass.
2. Whether Turbo mode is the default (with opt-out) or opt-in.
3. Rebrand depth: binary name only, or also config file names, env prefix, data directory, and
   server identity.
