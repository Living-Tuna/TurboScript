# TurboScript — implementation plan

Companion to [TOKEN-ANALYSIS.md](./TOKEN-ANALYSIS.md). This is the design review
document: what changes, where, in what order, and how we prove nothing broke. No
optimization ships wired on by default until its flag has been measured against a
real session.

## Ground rules

1. **History is never rewritten.** Aging, dedup, and budgeting are request-time
   views over durable history. `/compact`, revert, export, and the DB stay intact.
2. **Every optimization is behind a flag** (`packages/schema/src/config/flag.ts`)
   and a config block, so it can be turned off without a revert.
3. **One behavior per commit**, each independently revertable.
4. **New code in new files** where possible; call-site edits are a few lines. This
   keeps `git merge upstream/dev` cheap.
5. **Measure before defaulting.** Flag on in dev builds for testing, off in release
   until the benchmark shows the win.
6. **Do not touch**: package names (`@opencode/*`), plugin IDs, protocol identifiers,
   DB schema, or the 7.5k display strings. Those are merge hazards, not token costs.

## 1. Tool-result aging (4.1)

### Where it plugs in

`packages/core/src/session/runner/llm.ts` currently does:

```
loaded = context.load(...)
estimate = SessionCompaction.estimatePrompt(loaded)
compaction.compact({ reason: "auto", context: loaded, estimate })
transcript = SessionModelRequest.baseTranscript({ messages: loaded.messages, ... })
context.request.primary({ messages: transcript.messages, inputTokens: estimate, ... })
```

Aging becomes a view step:

```
loaded = context.load(...)
view = aging.apply(loaded)              // NEW: same shape, older tool results stubbed
estimate = SessionCompaction.estimatePrompt(view)
compaction.compact({ context: loaded, estimate })   // full history for the summary
transcript = baseTranscript({ messages: view.messages, ... })
```

Compaction still reads the unabridged `loaded`, so summaries never see stubs.
Estimates use `view`, so we do not compact early just because we pruned. Subagents
use the same runner, so they inherit the behavior per session.

### New module

`packages/core/src/session/aging.ts`

```
export type Settings = {
  keepResults: number      // newest N tool results always verbatim (default 6)
  minChars: number         // never stub results smaller than this (default 2_000)
  highWatermark: string|number // start aging above this share/token count (default "55%")
  lowWatermark: string|number  // age down to this (default "40%")
  minGapResults: number    // minimum new results between two cuts (default 8)
}
export interface Interface {
  readonly apply: (context: SessionContext.Loaded) => Effect.Effect<SessionContext.Loaded>
}
```

Pure core function (exported for tests):

```
ageToolResults(messages, settings, watermark) =>
  { messages, firstKeptID, stubbed, charsRemoved }
```

### Algorithm

1. Walk assistant messages in order, collect completed tool parts
   (`state.status === "completed"`) with their character cost.
2. Load the session watermark from KV: `turbo/age/<sessionID>` =
   `{ firstKeptID, resultCount }`.
3. Results before `firstKeptID` are always stubbed (recomputed, not persisted).
4. Advance the watermark only when:
   - estimated prompt > `highWatermark`, **and**
   - `resultCount - watermark.resultCount >= minGapResults` (or hard budget).
5. On advance, stub oldest results until the estimate drops below `lowWatermark`,
   never touching the newest `keepResults` nor results after the last user message.
6. Stub = one text part, call id/name/input preserved:

```
[elided read of src/app.ts, 1,840 lines; read again if needed]
[elided bash output, 22,400 chars; see tool_ab12cd34]
```

7. Persist the new `{ firstKeptID, resultCount }` after building the view.
8. Return a shallow copy of the messages that changed; everything else by reference.

### Why the watermark is stored

Prompt caching is prefix-based. If the stub boundary moved every step, every step
would invalidate the cache from that point on. The watermark makes cuts rare and
keeps the prefix byte-identical between them; hysteresis (`high` → `low`) and
`minGapResults` bound the cost of each invalidation.

Measured expectations: ~30–50% fewer prompt tokens in sessions longer than ~40
assistant messages; no effect on short sessions.

### Edge cases

| Case | Behavior |
| --- | --- |
| Result already stubbed | Reset `state.content` to the same stub deterministically |
| Error results | Kept verbatim when < `minChars`; otherwise stub keeps the first 200 chars |
| File/media parts in results | Kept if recent; `[elided image]` when aged |
| Session resumed in a new process | Watermark is in KV, so the view is identical |
| History replaced by compaction | Stored `firstKeptID` gone → start from newest `keepResults`, wait for gap |
| Retry of the same step | Same deterministic view |
| `measured` anchor older than a new cut | Estimate overstates slightly; conservative, never under-sends |

### Tests

`packages/core/test/session-aging.test.ts` (pure function, no services):

- keeps the newest N results verbatim
- stubs the oldest first and stops at the low watermark
- does nothing below the high watermark
- respects `minGapResults` and never re-cuts on the same step
- preserves call id, name, and input; stubs only result content
- keeps error results under `minChars`
- never stubs results after the last user message
- deterministic output for identical input

`packages/core/test/session-aging-service.test.ts`: KV watermark read/write,
compaction boundary reset, and `Loaded` round-trip through `estimatePrompt`.

## 2. Tighter output caps and read paging (4.2)

- `tool_output.max_lines` / `max_bytes` already exist and bound shell output
  (`packages/shell/src/shell.ts` reads `ToolOutput.MAX_LINES`). Change the
  **defaults** only under the Turbo profile: 400 lines / 12 KiB.
- `read` has independent constants (`MAX_READ_LINES`, `MAX_READ_BYTES` in
  `packages/core/src/tool/read-filesystem.ts`). Plumb them from
  `ConfigToolOutput` instead of hardcoding, and give the tool's truncation marker a
  `continue with offset: N` hint (it already emits one).
- No outline mode in the first pass. It needs a parser per language and the
  schema field adds prefix tokens; `grep` + paged `read` already work. Revisit after
  measuring.

Risk: agents page more on large files. Mitigation: keep the spill path and the
offset hint; measure tool-call counts, not just tokens.

## 3. Coding-only scope (4.4)

No core code needed: tool visibility is already permission-filtered
(`whollyDisabled(tool.options?.permission ?? name, rules)` in
`packages/core/src/tool.ts:281`), and denied tools disappear from the schema
entirely.

- Add a shipped default config under `packages/core/src/config/` (or the CLI's
  default global config) with a `coding` agent:
  - allow: `read`, `edit`, `write`, `patch`, `grep`, `glob`, `shell`, `todo`
  - deny: `websearch`, `webfetch`, `image_generate`, `video_generate`, `automation`,
    `artifact`, `question`, `skill`, `subagent`
  - `steps: 40` (upstream default is unlimited)
- Do not delete the tools; the profile only denies them. One config line restores
  any of them.
- MCP stays unconfigured by default (nothing to change; just do not add servers).

Expected: smaller schema prefix and fewer analysis detours. Measure prompt tokens
on the first request of a fresh session before/after.

## 4. Read dedup (4.3)

In `packages/core/src/tool/plugin/read.ts`, keep a location-scoped map
`(sessionID) → path → { mtimeMs, size, ranges: Set<"offset:limit">, ordinal }`.

- If the same range is requested again and `mtimeMs`/`size` are unchanged, return
  `[unchanged since step N: src/app.ts, 1,200 lines]` instead of content.
- Records expire with the session's location (in-memory only; a restart costs one
  full read).
- Never stub the first read of a path, and never stub when the file changed.

Test: two identical reads in one session → second is a stub; touch the file between
them → full content again.

## 5. Auxiliary model routing (4.5)

`packages/core/src/session/compaction.ts:275` (`selectModel`) already honors a
`compaction` agent model but falls back to the session model. Change the fallback to
`Model.small(providerID)` when the agent has none, mirroring
`SessionContext.selectTitle`, and fall back to the session model if the small model
is unavailable or the summary fails schema validation.

Also apply the same fallback to `generate` if it is not already small-model routed.
No change to title (already handled).

## 6. Instruction budget (4.6)

Only instruction files need this. Code-mode guidance is already conditional:
`codeModeCatalog` is `undefined` when Code Mode is off, and the instructions render
as `removed` (`packages/core/src/tool.ts:307`,
`packages/core/src/codemode/instructions.ts:150`).

- Add `experimental.turbo.instructions.max_bytes` (default off).
- In `Instructions` render (or `instruction-discovery.ts`), truncate each source at
  the budget with `[... truncated, read <path> for the rest]`, keeping the first
  part (headings and top rules) and dropping the tail.
- Never truncate a source below 2 KiB; skip truncation entirely for the global file.

## 7. Budget guard (4.7)

Session totals are already cumulative on the session row
(`tokens_input`, `tokens_output`, `tokens_cache_read`, `cost`; see
`packages/core/src/session/projector.ts:74`), so the check is one cheap row read.

- Config: `experimental.turbo.budget = { soft_tokens, hard_tokens, hard_cost }`.
- Soft: lower the aging high/low watermarks for this session (start trimming earlier).
- Hard: before a step, publish a session error
  (`SessionEvent.Step.Failed` with a clear message naming the limit and how to raise
  it) and stop the run. Never silently truncate.
- Defaults: none (opt-in), so upstream behavior is untouched until configured.

## 8. Rebrand

Full display rebrand is out of scope on purpose: 7,499 `OpenCode` occurrences across
the packages would make every upstream merge painful. Rebrand the identity layer and
the surfaces a user actually touches.

### Phase A — identity (safe, reversible)

| Surface | Where | Change |
| --- | --- | --- |
| App name for XDG dirs | `packages/util/src/global.ts` | `turboscript` (data/config/state/log) |
| Config names | `packages/core/src/config/discovery.ts` | `turbo.json(c)` first, `opencode.json(c)` still read as fallback |
| System config dir | same file | `/etc/turboscript`, `%ProgramData%\turboscript` |
| CLI command name | `packages/cli/script/build.ts` define | `OPENCODE_CLI_NAME = "turbo"` (dev default too) |
| Binaries | `packages/cli/package.json` | add `turbo`, `turboscript`; keep `opencode` alias |
| Env prefix | CLI entry shim | copy `TURBO_*` → `OPENCODE_*` when unset; avoids editing 155 call sites |
| TUI wordmark/help | TUI header + `--help` | `TurboScript` |
| Data dir notice | CLI startup | if `~/.local/share/turboscript` is new and an opencode dir exists, print a one-line hint (`turbo migrate`), never auto-move |

Add `packages/util/src/identity.ts` as the single source for these strings.

### Phase B — optional migration command

`turbo migrate` copies (not moves) config and the DB snapshot from the opencode
directories after confirmation. Off until the user runs it.

Not changed: npm package names, plugin IDs, protocol/SDK identifiers, release
channels.

## 9. Commit sequence

1. `docs: add token analysis and implementation plan`
2. `feat(core): add tool-result aging view` (module + tests, not wired)
3. `feat(core): wire tool-result aging behind turbo flag`
4. `feat(core): session token budget guard`
5. `feat(core): small-model compaction fallback`
6. `feat(core): read dedup for unchanged files`
7. `feat(core): configurable read limits`
8. `feat(config): coding-only turbo profile`
9. `feat(cli): turbo identity, config aliases, env shim`
10. `docs: turbo mode, migration, measurement`

## 10. Verification

- Per package: `bun test` from the package directory (never repo root);
  `bun typecheck` while iterating; `bun run check` at the root before pushing.
- Aging: unit tests plus one manual session on a real repo, comparing
  `tokens_input + tokens_cache_read` from the session row before/after.
- Benchmark: replay a recorded session's tool payloads through the view function and
  report prompt tokens per completed task. Add `turbo stats --tokens` afterwards.
- Rebrand: `bun run dev` (TUI starts, config discovered from `turbo.json`), `turbo --help`,
  and a fresh data dir. Confirm `opencode.json` still works.
- Upstream sync: `git fetch upstream dev && git diff upstream/dev --stat` should stay
  small; no formatting-only churn.
