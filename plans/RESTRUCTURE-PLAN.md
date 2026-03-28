# Plan: Merge nemoclaw-project into rfn-ventures + Workstream-Scoped Context System

## Context

rfn-ventures is a monorepo with 15+ workstreams (GBP, ARP, Devin, EOS, NemoClaw, etc.) but tracks state in two monolithic files: `.claude/context.md` (104KB) and `.claude/TODO.md` (~30KB). Every session and every workstream writes to the same files, causing:
- **Context pollution**: GBP sessions parse ARP and Devin state unnecessarily
- **Merge conflicts**: concurrent sessions or different workstreams touch the same files
- **Confusing restarts**: the RESUME PROMPT tries to summarize everything at once
- **nemoclaw-project** is a separate repo that should fold into rfn-ventures

## Approach: Per-Workstream Context + NemoClaw Merge

### New Structure

```
.claude/workstreams/
  PROJ-001-platform/context.md, todo.md   # Infra, cron, VPS, Brain
  PROJ-002-gbp/context.md, todo.md        # GBP content pipeline
  PROJ-003-arp/context.md, todo.md        # ARP/Instantly/GHL outreach
  PROJ-004-devin/context.md, todo.md      # Devin Brain + dashboard
  PROJ-005-koala-ops/context.md, todo.md  # Revenue ops, HCP, QBO
  PROJ-006-eos/context.md, todo.md        # EOS workflows, scorecard, L10
  PROJ-007-video/context.md, todo.md      # AI video factory, ComfyUI, LoRA
  PROJ-008-media-intel/context.md, todo.md # GA4, Google Ads, Local Falcon
  PROJ-009-energy/context.md, todo.md     # Home Energy Savings platform
  PROJ-010-nemoclaw/context.md, todo.md   # OpenClaw sandbox, Ollama, DGX Spark

infra/nemoclaw/                           # NEW — NemoClaw infra config
  openclaw-sandbox-ollama.yaml

docs/reference/nemoclaw-dgx-spark-reference.md    # NemoClaw reference doc
docs/adr/ADR-136-sandbox-to-host-ollama-networking.md  # NemoClaw ADR
```

The global `.claude/context.md` and `.claude/TODO.md` become slim routing hubs (~30 lines each) pointing to workstream-specific files. Cross-cutting items only (VPS health, n8n restart) stay global.

---

## Phase 1: Create workstream structure (no behavior change)

**Goal**: Create the per-workstream directories and decompose the monolithic files.

1. Create `.claude/workstreams/PROJ-{001..010}-{slug}/` directories
2. **Decompose** the current monolithic `.claude/context.md` into 10 workstream-specific context files by extracting relevant sections
3. **Decompose** the current monolithic `.claude/TODO.md` into 10 workstream-specific todo files by extracting relevant items
4. Commit: `docs: create per-workstream context/todo structure`

**Critical files to read before decomposing:**
- `.claude/context.md` (104KB — full session history, need to route each section to the right workstream)
- `.claude/TODO.md` (need to categorize each item by PROJ-NNN)
- `docs/portfolio.md` (has the PROJ-NNN registry to validate against)
- `docs/project-register.md` (same)

## Phase 2: Merge NemoClaw files

**Goal**: Copy nemoclaw-project assets into rfn-ventures.

The nemoclaw-project repo is at `KIAPN/nemoclaw-project` on GitHub (also known as `KIAPN/nemoclaw`). It contains 3 files:

| Source file | Destination in rfn-ventures |
|---|---|
| `NemoClaw-DGX-Spark-Reference.md` | `docs/reference/nemoclaw-dgx-spark-reference.md` |
| `docs/adr/001-sandbox-to-host-ollama-networking.md` | `docs/adr/ADR-136-sandbox-to-host-ollama-networking.md` |
| `policies/openclaw-sandbox-ollama.yaml` | `infra/nemoclaw/openclaw-sandbox-ollama.yaml` |

Steps:
1. Fetch the 3 files from GitHub or clone the repo temporarily
2. Copy files to their destinations, renumbering the ADR to 136
3. Update the ADR header (status, number) to match rfn-ventures convention
4. Seed `.claude/workstreams/PROJ-010-nemoclaw/context.md` with current state (see NemoClaw Status below)
5. Seed `.claude/workstreams/PROJ-010-nemoclaw/todo.md` with next steps (see NemoClaw TODOs below)
6. Create `docs/charters/PROJ-010-nemoclaw.md` per governance framework (ADR-134)
7. Update `docs/portfolio.md` and `docs/project-register.md` with PROJ-010
8. Commit: `feat(PROJ-010): merge nemoclaw-project into rfn-ventures`

### NemoClaw Status (for seeding PROJ-010 context.md)

As of 2026-03-28, NemoClaw is fully installed and working on DGX Spark (spark-9371):

- **Sandbox**: devin2 (Landlock + seccomp + netns), Phase: Ready
- **Inference**: ollama-local / nemotron-3-super:latest (120B, local on GB10)
- **Tested**: "how many r's in strawberry" answered correctly by both 4B and Super models
- **Policy**: v4 applied with ollama_local network policy for direct sandbox-to-host routing
- **Presets**: pypi, npm applied
- **UFW**: rule added for 172.18.0.0/16 to port 11434
- **Port forward**: 18789 backgrounded for OpenClaw UI
- **Chromium**: snap installed for local dashboard access
- **Onboard wizard**: completed WITHOUT NEMOCLAW_EXPERIMENTAL=1 (Ollama graduated to standard menu as of PR #648)
- **Key fix**: PR #1037 (Mar 28) fixed the inference.local HTTP 403 issue from previous sessions
- **Dashboard URL**: `http://127.0.0.1:18789/#token=<tokenized>` (token stored in Doppler)

Key technical facts:
- Host Ollama at 172.18.0.1:11434 from sandbox (NOT localhost)
- Do NOT use /v1 suffix when OpenClaw talks natively to Ollama
- API key placeholder: `local-ollama`
- Cold load of Super 120B: 50-74 seconds, warm: ~24 seconds
- Node.js v22.22.1 via nvm (minimum 22.16)
- `sudo env "PATH=$PATH" $(which npm)` required for npm commands under sudo (nvm not in sudo PATH)

### NemoClaw TODOs (for seeding PROJ-010 todo.md)

- [ ] Explore speculative decoding: pair nemotron-3-super (draft) with nemotron-3-nano:4b
- [ ] Replicate Nemotron Super on Brain PC (RTX 6000 Pro 96GB) with MTP/speculative decoding disabled
- [ ] Investigate Nemotron 3 Ultra (500B) multi-node setup when second Spark is available
- [ ] Evaluate Nemotron 3 Omni and VoiceChat when publicly released
- [ ] Configure persistent env vars in sandbox (NVIDIA_API_KEY, ANTHROPIC_API_KEY)
- [ ] Test agent tool use with Super (not just simple Q&A)
- [ ] Monitor PR #781 — local-inference policy preset (when merged, may replace custom policy)
- [ ] Update reference doc with 2026-03-28 session changes (NEMOCLAW_EXPERIMENTAL no longer needed, PR fixes)

## Phase 3: Replace global context/TODO with routing hubs

**Goal**: Slim the monolithic files now that workstream files exist.

1. Archive current files:
   - `.claude/context.md` → `.claude/archive/context-pre-restructure-2026-03.md`
   - `.claude/TODO.md` → `.claude/archive/TODO-pre-restructure-2026-03.md`
2. Replace `.claude/context.md` with a ~30 line routing hub:
   - Workstream index table (PROJ-NNN → slug → focus area)
   - Cross-cutting state only (VPS IP, Brain IP, cron status — under 10 lines)
   - Instruction: "Read ONLY the workstream context relevant to your current task"
3. Replace `.claude/TODO.md` with a ~20 line routing hub:
   - Cross-project items only (n8n restart, monthly re-score)
   - Ryan-blocked items with PROJ-NNN tags
   - Instruction: "Per-project TODOs are in `.claude/workstreams/PROJ-NNN-slug/todo.md`"
4. Remove the giant RESUME PROMPT from the global TODO — each workstream's context.md carries its own resume state
5. Commit: `docs: replace monolithic context/todo with routing hubs`

## Phase 4: Update commands and rules

**Goal**: Make the tooling workstream-aware.

1. **Update `.claude/commands/a_resume.md`**:
   - Read `docs/portfolio.md` for the workstream index
   - Prompt: "Which workstream? (or detect from branch prefix)"
   - Read only `.claude/workstreams/PROJ-NNN-slug/context.md` and `todo.md`
   - Present focused resume summary for that workstream only

2. **Update `.claude/commands/a_wrap.md`**:
   - Write to workstream-specific context.md and todo.md, not global
   - Commit only those workstream files: `git add .claude/workstreams/PROJ-NNN-slug/`

3. **Update `.claude/rules/push-context-sync.md`**:
   - Detect current workstream from branch prefix or changed files
   - Update only `.claude/workstreams/PROJ-NNN-slug/context.md` and `todo.md`
   - No longer touches the global routing hubs

4. **Update `.claude/rules/todo.md`**:
   - Write new todos to workstream-specific todo.md, not the global file
   - Cross-project items (rare) go to the global routing hub

5. **Update `.claude/hooks/session-start.sh`** (if it reads context.md):
   - Remove global context grep
   - Keep `portfolio.md` injection (lightweight, already works)

6. Commit: `feat: workstream-scoped context in commands and rules`

### Workstream Branch Convention

To enable automatic workstream detection in commands/rules:

| Branch prefix | Workstream |
|---|---|
| `platform/*`, `infra/*` | PROJ-001 |
| `gbp/*`, `content/*` | PROJ-002 |
| `arp/*`, `outreach/*`, `ghl/*` | PROJ-003 |
| `devin/*`, `dashboard/*` | PROJ-004 |
| `ops/*`, `koala/*` | PROJ-005 |
| `eos/*` | PROJ-006 |
| `video/*`, `lora/*` | PROJ-007 |
| `analytics/*` | PROJ-008 |
| `energy/*` | PROJ-009 |
| `nemoclaw/*`, `dgx/*` | PROJ-010 |
| `main` | Prompt user to declare workstream |

## Phase 5: Archive source repos

**Goal**: Clean up after validated merge.

1. Verify all nemoclaw content is in rfn-ventures and pushed
2. Archive `KIAPN/nemoclaw-docs` on GitHub: `gh repo archive KIAPN/nemoclaw-docs --yes`
3. Archive `KIAPN/NemoClaw` fork on GitHub: `gh repo archive KIAPN/NemoClaw --yes`
4. On DGX Spark: delete `~/nemoclaw-project`
5. On DGX Spark: clone rfn-ventures to `~/rfn-ventures` and pull latest

## Phase 6: Write ADR

1. Write `docs/adr/ADR-137-workstream-scoped-context-system.md`:
   - Why the monolithic context/todo system failed at scale
   - The per-workstream `.claude/workstreams/` solution
   - How commands, rules, and hooks were updated
   - The NemoClaw merge as part of the restructure

---

## Verification Checklist

After all phases, confirm:

- [ ] **Routing hubs are slim**: `.claude/context.md` < 50 lines, `.claude/TODO.md` < 30 lines
- [ ] **Each workstream has its own state**: 10 directories under `.claude/workstreams/` with populated context.md and todo.md
- [ ] **NemoClaw files present**: `docs/reference/nemoclaw-dgx-spark-reference.md`, `docs/adr/ADR-136-*`, `infra/nemoclaw/openclaw-sandbox-ollama.yaml`
- [ ] **Commands work**: `/resume` prompts for workstream and loads scoped context; `/wrap` writes to scoped files
- [ ] **No merge conflicts**: two sessions on different workstreams can push without touching the same files
- [ ] **Source repos archived**: nemoclaw-docs and NemoClaw fork archived on GitHub
- [ ] **Portfolio updated**: PROJ-010 appears in `docs/portfolio.md`
- [ ] **ADR written**: ADR-137 documents the decision

## Risks

| Risk | Mitigation |
|---|---|
| Claude forgets to scope and updates wrong context | `push-context-sync` rule explicitly names the workstream; routing hub says "DO NOT read all" |
| Session spans multiple workstreams | `/resume` can load multiple workstreams if asked; each is small (~1-3KB vs 104KB) |
| Migration breaks mid-session | Phase 1 creates new structure without changing behavior; old files still work until Phase 3 |
| WSL2 vs Spark path differences | rfn-ventures cloned to both machines; git handles sync |
