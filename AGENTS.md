# AGENTS.md — orlon-bot

> Behavioral contract for all AI agents working on this project.
> Read this file completely before writing a single line of code.

---

## 1. Project Identity

**Project:** `orlon-bot`
**What it is:** A personal style-tuned LLM — Llama 3.2 3B fine-tuned on Discord chat data using QLoRA (Unsloth), deployed via Ollama on `athena` (Radxa Rock 5T), exposed as a Telegram bot.
**Why it exists:** Close the ML pipeline gap for a DataSpark Werkstudent AI/Analytics application. The deliverable is a GitHub-hosted, documented end-to-end ML project.

---

## 2. Execution Model

### 2.1 Pipeline Stages (in order)
```
Stage 0 — Data          discord_parser.py → train.jsonl (DONE)
Stage 1 — Training      Kaggle/Colab: Unsloth + QLoRA → fine-tuned adapter
Stage 2 — Export        Merge adapter → GGUF → pull into Ollama on athena
Stage 3 — Tracking      MLflow server on athena logging all training runs
Stage 4 — Bot           python-telegram-bot on athena calling Ollama
Stage 5 — Eval          BLEU + human-eval comparison: base vs fine-tuned
Stage 6 — Docs          README + Notion Blueprint
```

Never skip stages. Never work on Stage N+1 before Stage N is tested end-to-end.

### 2.2 Hosting Map
| Component | Host | How |
|---|---|---|
| Training | Kaggle / Google Colab | Cloud GPU (free tier) |
| Model inference | `athena` (Rock 5T) | Ollama, port 11434 |
| MLflow tracking | `athena` | Docker Compose at `/srv/dev-shared/homelab/mlflow/` |
| Telegram bot | `athena` | pm2 process |
| Dev sync | `hermes` → `athena` | Mutagen (`radxa-sync`) |

### 2.3 Tailnet Access
- `athena` is on Tailscale: `athena.auxois-wyrm.ts.net`
- All services on `athena` bind to `127.0.0.1` only
- MLflow exposed via Caddy + `tailscale cert` file TLS if needed

---

## 3. Workflow Rules

1. **One stage at a time.** Complete, test, and log before moving on.
2. **No training without validated JSONL.** Run `python discord_parser.py --validate` before any Kaggle run.
3. **Every Kaggle run gets an MLflow entry.** Log: model, base, lr, epochs, loss curve, sample outputs.
4. **GGUF export is non-negotiable.** Training output must be converted to GGUF and tested in Ollama before declaring Stage 2 done.
5. **Bot never auto-sends without a safety filter pass.** Risky keywords (bank, password, OTP, urgent) → always route to draft mode.
6. **All Docker services on `athena` follow the standard layout:**
   ```
   /srv/dev-shared/homelab/<service>/
   ├── docker-compose.yml
   ├── data/
   └── config/
   ```
   Always `cd` into the service directory before `docker compose` commands.
7. **All ports bound to `127.0.0.1`** in docker-compose.yml — never `0.0.0.0`.

---

## 4. Coding Standards

### Python
- **Runtime:** Python 3.11+ managed via `mise` on both `hermes` and `athena`
- **Dependency management:** `uv` only — no pip install in bare environment
- **Style:** Black-formatted, type hints on all function signatures
- **No bare `except:`** — always catch specific exceptions
- **Scripts are self-contained** — each script has a `if __name__ == "__main__":` block and argparse

### Data Pipeline
- Input files always in `data/raw/` — never modify raw files
- Processed output always in `data/processed/`
- Parser must support multiple input files: `--input f1.txt f2.txt`
- All JSONL output validated before use

### Training (Kaggle/Colab)
- Notebooks are versioned in `training/` as `.ipynb`
- All hyperparams at the top of the notebook in a single config cell
- Save adapter checkpoint after every epoch
- Log to MLflow via `mlflow.log_params()` and `mlflow.log_metrics()`

### Telegram Bot (`bot/`)
- Single entry point: `bot/main.py`
- Config via `.env` file (never hardcoded tokens)
- Graceful shutdown on SIGTERM (pm2 compatibility)
- Draft mode default; auto-reply only on explicit opt-in flag

---

## 5. DEVLOG Hard Rule

**Every session ends with a DEVLOG entry. No exceptions.**

A session that produces no DEVLOG entry is considered incomplete.
The next agent picking up this project MUST read the last 3 DEVLOG entries before doing anything.

Format: `tasks/DEVLOG.md` — append-only, newest entry at bottom.

```
## YYYY-MM-DD — [Stage N: Short Title]

### What happened
- Bullet summary of work done

### What works
- Confirmed working items

### What's broken / blocked
- Known issues or blockers

### Next session starts here
- Exact first action for next session
```

---

## 6. Kickoff Template

Copy-paste this at the start of every new agent session:

```
Project: orlon-bot
Stage: [current stage number and name]
Last DEVLOG: [date of last entry]

Read: AGENTS.md, CONTEXT.md, tasks/DEVLOG.md (last 3 entries), tasks/todo.md
Then: [first action from last DEVLOG "Next session starts here"]
```

---

## 7. Out of Scope

The following are NOT part of this project:
- WhatsApp integration (legal risk)
- Discord self-bot / automating Vishal's personal Discord account
- Multi-model A/B testing (resume project, not production system)
- Retraining automation / continuous learning loop
- Public deployment (Tailscale-only, private)
