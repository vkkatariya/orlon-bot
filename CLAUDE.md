# CLAUDE.md
> This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.
> Project-specific instructions. Workflow conventions live in `./workflow/`.

---

## Session startup

Before any meaningful work, read these files in order:

1. `CONTEXT.md` — pipeline stages, infrastructure, data, secrets, hard constraints
2. `tasks/DEVLOG.md` (last 3 entries) — current world state + next-session start
3. `tasks/todo.md` — phased task list
4. `tasks/lessons.md` — active prevention rules

**This project uses the dual-session model** (see `workflow/SESSION-WORKFLOW.md` v3):

- **Local session** (this is normally the one you are): `[orlon-bot]-local`
  - Host: athena tmux
  - Process: Claude CLI on athena, started with `claude --remote-control '[orlon-bot]-local'`
  - Branch: whatever the working dir has checked out (work on `feat/<task>` sub-branches off `dev` or `main`)
  - Filesystem: full local access (athena's `/home/radxa/...`, Docker, etc.)
  - Use for: design iteration, file edits, debugging, dev server, anything needing local FS, deploys
- **Cloud session** (ephemeral, for heavy work): `[orlon-bot]-cloud`
  - Host: Anthropic cloud container (NOT your machine)
  - Process: started from https://claude.ai/code → Code tab → New session → pick repo → pick the sub-branch the local session is working on
  - Branch: same as the local session (both share the working dir on the repo; no separate lineage branches)
  - Filesystem: **ONLY the GitHub repo** (no `~/dev-shared/`, no Docker, no `.env.local`, no local servers)
  - Use for: `pnpm install` / `npm install`, Playwright runs, full builds, audits, anything needing CPU isolation
  - **Cannot do:** `vercel deploy` (no auth), access local servers, run Docker, see homelab configs

**No branch setup needed.** Both sessions share `dev` (or `main`) and work on `feat/<task>` sub-branches directly. There are no `claude/local` / `claude/cloud` lineage branches — that pattern was tried and removed (see L-069 in `tasks/lessons.md`).

**How to start the cloud session:** open https://claude.ai/code → Code tab → New session → pick the repo → pick the sub-branch the local session is working on → rename to `[orlon-bot]-cloud`. The cloud session spins up in an Anthropic container with a fresh clone.

**Cross-session coordination:** via git (both sessions commit to the same repo) and `tasks/DEVLOG.md` (the local session writes; the cloud session reads if it needs context, writes if it does heavy work). Cloud session work is committed to whatever sub-branch it's working on, then merged via PR like any other contribution.

 read top 3 of `tasks/DEVLOG.md` on every resume — the cloud and local sessions log to the same DEVLOG with `cloud-session-start` / `cloud-session-end` / `local-session-handoff` markers so the other side knows what happened. Coordination also happens via git branches and PRs.

## Workflow references (symlinked, homelab-only)

The `./workflow/` directory is a symlink to `~/dev-shared/workflow/` — same path on every machine via mutagen sync. **Do not commit it** (already in `.gitignore`). Read workflow files on demand, not at every session start:

- `./workflow/SESSION-WORKFLOW.md` — Claude Code session lifecycle, dual-session (local relay + Anthropic cloud), lineage branches, /remote-control, compaction
- `./workflow/AI-ROUTING.md` — L1/L2/L3 layer model, tool vs agent routing
- `./workflow/GIT-GITHUB-BLUEPRINT.md` — branch/commit/PR conventions
- `./workflow/agents_workflow/AI-AGENTS-ORCHESTRATION.md` — sub-agent dispatch patterns

If the symlink is broken on a fresh clone, recreate it:
```bash
ln -sf ~/dev-shared/workflow ./workflow
```

## Compaction + /remote-control (project-specific)

- **/remote-control** is the real command (not `/rc` — that's hallucinated)
- **Before manual /compact:** commit, push, append your work-in-progress to `tasks/DEVLOG.md`
- **After /compact** or /clear: re-read top 3 of `tasks/DEVLOG.md`, re-read this CLAUDE.md, check `git status` to reconstruct in-flight work
- **Pro plan 5hr rolling limit:** dual session uses 2 surfaces in parallel. Local for code edits + bot dev; cloud for validation/audit only (cloud can't run the pipeline)

---

## Commands

### Python setup
```bash
# Python 3.11+ via mise
uv venv && source .venv/bin/activate
uv pip install -r bot/requirements.txt
```

### Data pipeline
```bash
# Drop Discord .txt exports in data/raw/, then:
python data/discord_parser.py \
  --input data/raw/*.txt \
  --my-username ordinary_835 \
  --output data/processed/train.jsonl \
  --max-examples 20000

# Validate before any training:
python data/discord_parser.py --validate
```

### Training (Kaggle/Colab only — no GPU locally)
- Open `training/kaggle_notebook.ipynb` in Kaggle or Colab
- All training runs log to MLflow (params, metrics, artifacts)
- Save adapter checkpoint after every epoch
- Output: LoRA adapter + merged GGUF

### Deployment
```bash
# Ollama on athena (already running, port 11434)
docker compose up -d                          # MLflow
pm2 start bot/main.py --name orlon-bot         # Telegram bot
# Health: pm2 status; logs: pm2 logs orlon-bot
```

### Secrets
```bash
cp bot/.env.example bot/.env
# Edit bot/.env with: TELEGRAM_BOT_TOKEN, MLFLOW_TRACKING_URI, OLLAMA_HOST, OLLAMA_MODEL
# .env is gitignored — never commit
```

### CI
- Workflow: `.github/workflows/ci.yml`
- Runs: lint (ruff) + typecheck (mypy) + test
- **Does NOT train** — training requires Kaggle/Colab

---

## Architecture

### Current state: Stage 0 (data pipeline) DONE — Stage 1 (Kaggle training) NEXT

A personal style-tuned LLM (Llama 3.2 3B, QLoRA via Unsloth) trained on Discord chat data. Telegram bot exposes it. Built as a portfolio ML project for DataSpark Werkstudent AI/Analytics application.

### Pipeline stages (in order)
```
Stage 0 — Data          discord_parser.py → train.jsonl (DONE)
Stage 1 — Training      Kaggle/Colab: Unsloth + QLoRA → fine-tuned adapter
Stage 2 — Export        Merge adapter → GGUF → pull into Ollama on athena
Stage 3 — Tracking      MLflow server on athena logging all training runs
Stage 4 — Bot           python-telegram-bot on athena calling Ollama
Stage 5 — Eval          BLEU + human-eval comparison
Stage 6 — Docs          README + Notion Blueprint
```

**Never skip stages. Never work on Stage N+1 before Stage N is tested end-to-end.**

### Hosting map
| Component | Host | How |
|---|---|---|
| Training | Kaggle / Colab | Cloud GPU (free tier, 30h/week) |
| Model inference | `athena` (Rock 5T) | Ollama, port 11434 |
| MLflow tracking | `athena` | Docker Compose at `/srv/dev-shared/homelab/mlflow/` |
| Telegram bot | `athena` | pm2 process |
| Dev sync | `hermes` → `athena` | Mutagen (`radxa-sync`) |

### Directory layout
```
orlon-bot/
├── data/
│   ├── raw/                ← drop Discord .txt exports (gitignored)
│   ├── processed/          ← train.jsonl, val.jsonl (gitignored)
│   └── discord_parser.py
├── training/
│   ├── kaggle_notebook.ipynb
│   └── configs/train_config.yaml
├── deployment/
│   ├── docker-compose.yml   # MLflow
│   └── Modelfile            # Ollama custom model
├── bot/
│   ├── main.py              # python-telegram-bot
│   ├── requirements.txt
│   └── .env.example
├── eval/eval.py             # BLEU + sample comparison
└── tasks/                   # DEVLOG.md, todo.md, lessons.md
```

### Branch strategy
- `main` — trunk
- `feat/<slug>` — feature branches off `main`
- Conventional commits: `feat:`, `fix:`, `docs:`, `chore:`, `refactor:`

### Docker path convention (athena)
```
/srv/dev-shared/homelab/mlflow/
├── docker-compose.yml
├── data/      (bind mount)
└── config/
```

---

## Hard constraints

- **One stage at a time** — complete, test, and log before moving on
- **No training without validated JSONL** — run `python discord_parser.py --validate` before any Kaggle run
- **Every Kaggle run gets an MLflow entry** — log model, base, lr, epochs, loss curve, sample outputs
- **GGUF export is non-negotiable** — training output must be converted to GGUF and tested in Ollama before declaring Stage 2 done
- **Bot never auto-sends without a safety filter pass** — risky keywords (bank, password, OTP, urgent) → always draft mode
- **All Docker services on `athena` bind to `127.0.0.1`** — never `0.0.0.0`
- **PII in raw data is redacted** (phone, email, IBAN → `<REDACTED>`) by `discord_parser.py`
- **Never commit raw chat exports, train.jsonl, or model weights** (gitignored)
- **Never change trigger command strings** for any UI-facing surface — they're user-facing identifiers
- **Changing the Groq/Llama model requires a DEVLOG note** — performance-critical dependency

---

## Out of scope

- WhatsApp integration (legal risk)
- Discord self-bot / automating Vishal's personal Discord account
- Multi-model A/B testing (resume project, not production)
- Retraining automation / continuous learning loop
- Public deployment (Tailscale-only, private)

---

## Task management

Agents write to `tasks/DEVLOG.md` (newest at top) at the end of every session. Format and required fields are in `AGENTS.md`. This is mandatory — a missing DEVLOG entry breaks the next session's handoff.

Lessons from corrections go in `tasks/lessons.md` (newest at top, numbered L-0NN).

The `ref:resources/` directory exists in this project — note the colon in the name can confuse some git commands; use `--find-renames` when moving files in/out.
