# Tasks: Multi-Agent Agent X-Ray

**Epic**: Support Claude Code, Codex, Cursor, and AGY in one live session X-ray  
**Target**: Finish Cursor parity + harden adapters, then ship  
**Total Estimate**: ~9d effort, ~6d calendar (critical path)  
**Owner**: TBD  
**Status**: in-progress (large uncommitted local diff on `main`)

## Dependency Graph

```
[1.1 Shared --agent plumbing] ──→ [1.2 Cursor discovery/match] ──→ [1.3 Cursor adapters]
         │                                      │                         │
         │                                      └─────────────────────────┼──→ [1.4 Cursor context/skills/feed]
         │                                                                │
         ├──→ [2.1 Codex adapter review/harden]                           │
         ├──→ [2.2 AGY adapter review/harden]                             │
         └────────────────────────────────────────────────────────────────┴──→ [3.1 Tests]
                                                                              └──→ [3.2 Docs + release]
```

## Progress

```
Epic: Multi-Agent Agent X-Ray
Progress: ████████░░░░ 65% (roughly 8/12 task units done locally, not committed)
On track: ⚠️ At risk — large uncommitted WIP; Cursor model still missing; no automated tests
```

### Already done (local WIP — needs commit/PR)
- [x] `[feat]` Multi-agent discovery (`--agent`, session_files for claude/codex/cursor/agy)
- [x] `[bug]` Follow-mode ignored `--agent` (switched to AGY)
- [x] `[bug]` Cursor project path matching (`inov.es` / `claude-xray` slug encoding)
- [x] `[feat]` Cursor context history + approximate “what's in context”
- [x] `[bug]` DuplicateKey crash when multiple Cursor subagents run
- [x] `[feat]` Cursor skill detection (attached / `/command` / SKILL.md reads)
- [x] `[bug]` Activity timestamps + newest-first sort for Cursor stamps
- [x] `[bug]` Cursor subagents stuck “running” after `turn_ended` / abort

### Still open
- [ ] Cursor model display (shows `?` / empty today)
- [ ] Codex/AGY feature-parity hardening + mis-detect review
- [ ] Regression tests
- [ ] Docs polish + commit/PR

---

## Wave 1 (foundation — mostly done)

### [feat] Shared `--agent` plumbing
**Parent**: Multi-Agent Agent X-Ray  
**Priority**: P0  
**Estimate**: 1d (S) — **done locally**  
**Status**: review  
**Labels**: [cli] [session-discovery]

#### Description
Wire `--agent {auto,claude,codex,cursor,agy}` through discovery, picker, follow mode, and resolve.

#### Acceptance Criteria
- [x] `--agent cursor` never follows Claude/Codex/AGY sessions
- [x] `tick()` / `open_newest()` / picker / `--once` all honor agent filter
- [x] Picker shows agent column and filter label

#### Definition of Done
- [x] Code written and self-reviewed
- [ ] Tests passing
- [ ] PR created and approved
- [ ] Documentation updated

---

### [bug] Cursor project matching
**Parent**: Multi-Agent Agent X-Ray  
**Priority**: P0  
**Estimate**: 0.5d (XS) — **done locally**  
**Status**: review  
**Labels**: [cursor] [discovery]

#### Description
Cursor folder names encode `/` and `.` as `-` (lossy). Naive decode broke `inov.es/claude-xray`.

#### Acceptance Criteria
- [x] `cursor_slug` + `cursor_project_matches` find this repo’s transcripts
- [x] Numeric/`empty` project dirs no longer match every cwd

---

## Wave 2 (Cursor parity)

### [feat] Cursor live adapter (transcript → UI)
**Parent**: Multi-Agent Agent X-Ray  
**Priority**: P0  
**Estimate**: 2d (M) — **mostly done locally**  
**Status**: in-progress  
**Labels**: [cursor] [adapter]

#### Description
Parse `agent-transcripts` + `subagents/*.jsonl` into context, agents, skills, activity.

#### Acceptance Criteria
- [x] Main + subagent tool feed
- [x] Subagent finished on `turn_ended` / idle
- [x] Skills: attached, `/command`, SKILL.md reads
- [x] Activity shows date+time, newest first
- [ ] **Model name shown** (currently empty / `?`)
- [ ] Subagent descriptions filled from Task/`CallDynamicTool` prompts when possible

#### Technical Notes
- Agent transcripts lack API usage / model fields.
- Model lives in `~/.cursor/chats/<workspace>/<session>/`:
  - `store.db` blobs: `providerOptions.cursor.modelName` (e.g. `cursor-grok-4.5-high`)
  - `meta` hex JSON: `lastUsedModel` (often `"default"` — weak)
- Prefer latest `modelName` from store.db blobs; fall back to meta / “Composer”.

#### Dependencies
- Blocked by: none (discovery done)
- Blocks: tests, docs

---

### [feat] Cursor “what's in context” approximation
**Parent**: Multi-Agent Agent X-Ray  
**Priority**: P1  
**Estimate**: 1d (S) — **done locally**  
**Status**: review  
**Labels**: [cursor] [context]

#### Description
Rebuild context breakdown from logged chat turns when Claude’s request log is absent.

#### Acceptance Criteria
- [x] `d` / Enter opens approximate view for Cursor
- [x] User prompts / assistant replies / tool calls categorized
- [x] Footer clarifies missing system/tools/results

---

### [bug] Cursor model missing in header/context
**Parent**: Multi-Agent Agent X-Ray  
**Priority**: P1  
**Estimate**: 1d (S)  
**Status**: ready  
**Labels**: [cursor] [model]

#### Description
UI shows `model ?` for Cursor. Resolve model from Cursor chat store (or other stable source) and set `Session.model`.

#### Acceptance Criteria
- [ ] Context line shows real model id (e.g. `cursor-grok-4.5-high` or friendly name)
- [ ] Works with `--once` and TUI
- [ ] No crash if `store.db` / chat folder missing
- [ ] Read-only access only

#### Technical Notes
- Map session id → `~/.cursor/chats/*/<session-id>/store.db`
- Scan recent blobs for `"modelName":"..."`; cache on Session
- Optional: also set from system prompt “powered by Composer” as weak fallback

#### Dependencies
- Blocked by: none
- Blocks: accurate context-window inference for Cursor

#### Definition of Done
- [ ] Code written and self-reviewed
- [ ] Manual check on current session
- [ ] No new linting errors

---

## Wave 2b (other agents — parallelizable)

### [chore] Codex adapter harden + review
**Parent**: Multi-Agent Agent X-Ray  
**Priority**: P2  
**Estimate**: 1d (S)  
**Status**: backlog  
**Labels**: [codex] [adapter]

#### Description
Validate Codex event shapes against real sessions; fix cwd filter / context / feed gaps that look like bugs.

#### Acceptance Criteria
- [ ] `--agent codex` lists expected sessions for a known project
- [ ] Context / activity populate without crashes
- [ ] Known limitations documented (vs Claude)

---

### [chore] AGY adapter harden + review
**Parent**: Multi-Agent Agent X-Ray  
**Priority**: P2  
**Estimate**: 1d (S)  
**Status**: backlog  
**Labels**: [agy] [adapter]

#### Description
Review `transcript.jsonl` heuristic mis-detect risk; brain + antigravity-cli paths; picker labels.

#### Acceptance Criteria
- [ ] Non-AGY `transcript.jsonl` paths are not mislabeled when possible
- [ ] `--agent agy` works for Antigravity + CLI brain layouts
- [ ] Limitations documented

---

## Wave 3 (ship)

### [test] Regression coverage for multi-agent discovery + Cursor parsing
**Parent**: Multi-Agent Agent X-Ray  
**Priority**: P1  
**Estimate**: 2d (M)  
**Status**: backlog  
**Labels**: [test] [cursor]

#### Description
Fixture-based tests for slug matching, agent filter in follow mode, Cursor skill/activity/model parsing, DuplicateKey-safe feed keys.

#### Acceptance Criteria
- [ ] Unit tests for `cursor_slug` / `cursor_project_matches`
- [ ] Unit tests for feed key uniqueness across subagents
- [ ] Fixture transcript → Session assertions (skills, finished agents, model if fixture includes store)
- [ ] `uv run pytest` (or chosen runner) green

#### Dependencies
- Blocked by: Cursor model task (or mock fixtures without store.db)
- Blocks: release confidence

---

### [docs] README + usage for multi-agent
**Parent**: Multi-Agent Agent X-Ray  
**Priority**: P2  
**Estimate**: 0.5d (XS)  
**Status**: ready (partially done)  
**Labels**: [docs]

#### Description
Document agent filters, Cursor limitations (approx context, no tool results, model source), and local run via `uv run`.

#### Acceptance Criteria
- [x] README mentions Claude/Codex/Cursor/AGY
- [ ] Cursor limitations called out (context approx, model from chat store)
- [ ] “Develop from this repo” section: `uv run claude-xray --agent cursor`

---

### [chore] Commit / PR the multi-agent WIP
**Parent**: Multi-Agent Agent X-Ray  
**Priority**: P0  
**Estimate**: 0.5d (XS)  
**Status**: blocked  
**Labels**: [release]

#### Description
Split or ship the large uncommitted `claude_xray.py` + README diff once P0 Cursor gaps are acceptable.

#### Acceptance Criteria
- [ ] Clean commit message(s) focused on why
- [ ] No secrets in diff (`store.db` keys never written)
- [ ] PR summary + test plan

#### Dependencies
- Blocked by: Cursor model display (P1) **or** explicit decision to ship without it
- Blocks: external users getting fixes

---

## Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Cursor transcript schema drifts | High | Fixture tests; defensive parsers |
| Chat `store.db` encrypted / inaccessible | Medium | Fall back to `?`; never require decryption key for core UI |
| Large monolithic uncommitted diff | High | Commit in waves; don’t wait for perfect AGY/Codex parity |
| False-positive slash “skills” | Low | Keep path exclusions; prefer `manually_attached_skills` |

## Open Questions

1. Ship Cursor model as raw `cursor-grok-4.5-high` or map to a short label?
2. Commit now without model, or block the PR on model display?
3. Are Codex/AGY in scope for this PR, or Cursor-only hardening first?

## Suggested Next Sprint Focus

1. **[P1] Cursor model from `~/.cursor/chats/.../store.db`**
2. **[P0] Commit/PR** current Cursor fixes
3. **[P1] Minimal regression tests** for discovery + feed keys + skills
