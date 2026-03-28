# Handoff: Execute rfn-ventures Restructure + NemoClaw Merge

## Instructions for Claude Code on WSL2

You are picking up a pre-approved plan to restructure the rfn-ventures monorepo. The plan was designed and approved by Ryan on DGX Spark (spark-9371) on 2026-03-28. Execute it from the WSL2 rfn-ventures working copy.

**Read the full plan first**: This repo's `plans/RESTRUCTURE-PLAN.md` contains the complete, approved plan with 6 phases. Read it entirely before starting.

**Key context you need:**
- rfn-ventures has 15+ workstreams sharing monolithic `.claude/context.md` (104KB) and `.claude/TODO.md` (~30KB)
- These files cause merge conflicts, context pollution, and confusing restarts
- The fix: per-workstream context/todo files in `.claude/workstreams/PROJ-NNN-slug/`
- Additionally, this repo (nemoclaw-project / KIAPN/nemoclaw) needs to be merged INTO rfn-ventures and then archived

## Execution Order

### Step 1: Read the plan
```
Read plans/RESTRUCTURE-PLAN.md in this repo
```

### Step 2: Switch to rfn-ventures
```
cd ~/path/to/rfn-ventures  (wherever the WSL2 clone lives)
git pull
```

### Step 3: Execute Phases 1-6 in order
Each phase has a separate commit. Do not batch multiple phases into one commit.

**Phase 1** is the most labor-intensive — decomposing 104KB of context.md and TODO.md into 10 workstream-specific files. Read the monolithic files carefully and categorize each section/item by workstream. When in doubt about which workstream an item belongs to, use these hints:

| Keywords in the item | Workstream |
|---|---|
| cron, VPS, SSH, Docker, n8n restart, monorepo, backup | PROJ-001 platform |
| GBP, content pipeline, approval email, publish, Gemini strategist, pest_control diversity | PROJ-002 gbp |
| ARP, Instantly, GHL, outreach, DNC, LOB, canary, enrollment | PROJ-003 arp |
| Devin, dashboard, email adapter, Supabase, triage, brain agent | PROJ-004 devin |
| HCP, QBO, QuickBooks, job profitability, unscheduled jobs | PROJ-005 koala-ops |
| scorecard, L10, rocks, IDS, people analyzer, three-strike, EOS tracker | PROJ-006 eos |
| video, ComfyUI, LoRA, assembly, Wan, fal.ai, OpusClip | PROJ-007 video |
| GA4, Google Ads, Local Falcon, Zoom calls, weather, analytics | PROJ-008 media-intel |
| energy savings, HERS, energy audit | PROJ-009 energy |
| NemoClaw, OpenClaw, Ollama, sandbox, DGX Spark, Nemotron | PROJ-010 nemoclaw |

**Phase 2** fetches nemoclaw files from GitHub. Use:
```bash
gh api repos/KIAPN/nemoclaw/contents/NemoClaw-DGX-Spark-Reference.md -q '.content' | base64 -d > docs/reference/nemoclaw-dgx-spark-reference.md
gh api repos/KIAPN/nemoclaw/contents/docs/adr/001-sandbox-to-host-ollama-networking.md -q '.content' | base64 -d > docs/adr/ADR-136-sandbox-to-host-ollama-networking.md
gh api repos/KIAPN/nemoclaw/contents/policies/openclaw-sandbox-ollama.yaml -q '.content' | base64 -d > infra/nemoclaw/openclaw-sandbox-ollama.yaml
```
Then update the ADR header (number → 136, add rfn-ventures convention fields).
The NemoClaw context and TODO content is provided in the plan under "NemoClaw Status" and "NemoClaw TODOs".

**Phase 3** archives the old monolithic files and replaces them with slim routing hubs. Be careful — the archived copies must be complete. `cp` first, then overwrite.

**Phase 4** updates 4-5 files in `.claude/commands/` and `.claude/rules/`. Read each file first, understand the current behavior, then modify to be workstream-aware. Don't break the existing command structure — just scope the context/todo reads and writes.

**Phase 5** archives repos via `gh`. Only do this AFTER pushing all rfn-ventures changes and verifying.

**Phase 6** writes the ADR documenting the decision.

### Step 4: Push and verify
After all phases, push to GitHub and run the verification checklist from the plan.

### Step 5: Notify Ryan
When done, the DGX Spark session will:
1. `git clone` rfn-ventures to `~/rfn-ventures`
2. Delete `~/nemoclaw-project`
3. Verify the new structure works with `/resume`

## Important Notes

- **Do NOT modify** `.claude/rules/`, `.claude/commands/`, `.claude/skills/`, `.claude/hooks/` files other than the specific ones listed in Phase 4
- **Preserve all existing rules and hooks** — only modify the context/todo scoping behavior
- **Each phase gets its own commit** — this makes rollback easy if something breaks
- **The 10 workstream slugs are confirmed** — do not add, remove, or rename them
- **Read `docs/portfolio.md` and `docs/project-register.md`** to understand the existing PROJ-NNN registry before adding PROJ-010
- **Read `docs/adr/ADR-134-project-governance-framework.md`** before creating the PROJ-010 charter
