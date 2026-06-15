# CONTEXT.md — orlon-bot

> Project-specific context. Not behavioral rules (see AGENTS.md).
> Update this file whenever infrastructure or project state changes.

---

## 1. What This Project Is

A personal LLM clone pipeline built as a resume/portfolio ML project.

**Goal:** Demonstrate end-to-end ML pipeline competency for a DataSpark Werkstudent AI/Analytics application.

**Deliverable:** GitHub repo with documented pipeline: data preprocessing → QLoRA fine-tuning → MLflow tracking → GGUF/Ollama deployment → Telegram Bot API integration.

**NOT a production system.** Private, Tailscale-only access. The bot replies "in Vishal's style" — it's a style-tuned assistant, not an identity clone.

---

## 2. Infrastructure

### athena (primary node — all runtime services)
- **Hardware:** Radxa Rock 5T, ARM64, 24 GB RAM, 6 TOPS NPU
- **OS:** Armbian + Docker
- **Tailscale hostname:** `athena.auxois-wyrm.ts.net`
- **Role:** Ollama inference, MLflow tracking server, Telegram bot (pm2)
- **Dev sync:** Mutagen `radxa-sync` active from `hermes`

### hermes (dev workstation)
- **Hardware:** Mac (M1 Max)
- **Role:** Code editing, Kaggle notebook prep, Mutagen source
- **Unsloth Studio:** installed (MPS — inference only, not training)

### Kaggle / Google Colab
- **Role:** Training only — CUDA GPU (T4/P100)
- **Kaggle:** free 30h GPU/week quota
- **Output:** fine-tuned LoRA adapter + merged GGUF

### GitHub
- **Repo:** `vkkatariya/orlon-bot` (to be created)
- **What goes in:** all code, notebooks, README, eval results
- **What does NOT go in:** raw chat exports, train.jsonl (PII), model weights

---

## 3. Stack

| Layer | Tool | Version / Notes |
|---|---|---|
| Language | Python | 3.11+, managed via `mise` |
| Dependency mgmt | `uv` | only — no bare pip |
| Data pipeline | `discord_parser.py` | custom, in `data/` |
| Fine-tuning | Unsloth + QLoRA | Kaggle/Colab, CUDA |
| Base model | Llama 3.2 3B Instruct | `unsloth/Llama-3.2-3B-Instruct` |
| Experiment tracking | MLflow | Docker on athena |
| Inference runtime | Ollama | on athena, port 11434 |
| Bot framework | python-telegram-bot | v21+ |
| Process manager | pm2 | bot process on athena |
| Version control | Git + GitHub | repo: `vkkatariya/orlon-bot` |

---

## 4. Directory Structure

```
orlon-bot/
├── AGENTS.md                  # behavioral rules for AI agents
├── CONTEXT.md                 # this file
├── README.md                  # public-facing project README
│
├── data/
│   ├── raw/                   # ← drop Discord .txt exports here (gitignored)
│   ├── processed/             # train.jsonl, val.jsonl (gitignored)
│   └── discord_parser.py      # raw → JSONL pipeline
│
├── training/
│   ├── kaggle_notebook.ipynb  # Unsloth QLoRA fine-tune notebook
│   └── configs/
│       └── train_config.yaml  # hyperparams
│
├── deployment/
│   ├── docker-compose.yml     # MLflow on athena
│   └── Modelfile              # Ollama Modelfile for custom model
│
├── bot/
│   ├── main.py                # Telegram bot entry point
│   ├── requirements.txt
│   └── .env.example
│
├── eval/
│   └── eval.py                # BLEU + sample comparison script
│
└── tasks/
    ├── DEVLOG.md
    ├── todo.md
    └── lessons.md
```

---

## 5. Current State (as of 2026-06-07)

| Stage | Status | Notes |
|---|---|---|
| 0 — Data pipeline | ✅ Done | `discord_parser.py` written and tested |
| 0 — Training data | ✅ Ready | 20,000 pairs from 1 Discord DM export |
| 1 — Kaggle notebook | ⬜ Next | Unsloth Llama 3.2 3B QLoRA |
| 2 — GGUF export | ⬜ Waiting | Depends on Stage 1 |
| 3 — MLflow on athena | ⬜ Waiting | Docker Compose setup needed |
| 4 — Telegram bot | ⬜ Waiting | Depends on Stage 2 |
| 5 — Eval | ⬜ Waiting | Depends on Stage 4 |
| 6 — README + GitHub | ⬜ Waiting | Final polish |

---

## 6. Data Notes

### Training data
- **Source:** Discord DM export (`.txt`, DiscordChatExporter format)
- **Volume:** 143,129 raw messages → 20,000 quality training pairs (after merge + filter)
- **Language:** English only
- **Quality:** Good — clear sentences, consistent style
- **Limitation:** Single conversation (1 person). More exports = better generalization.
- **PII:** Redacted by parser (phone numbers, emails, IBANs → `<REDACTED>`)

### Adding more data
```bash
# Add more .txt exports to data/raw/, then re-run:
python data/discord_parser.py \
  --input data/raw/*.txt \
  --my-username ordinary_835 \
  --output data/processed/train.jsonl \
  --max-examples 20000
```

---

## 7. Key Decisions & Rationale

| Decision | Rationale |
|---|---|
| Telegram over WhatsApp/Discord | Official Bot API, no ToS risk, simplest integration |
| Llama 3.2 3B over Phi-3 mini | Better instruction-tuning, first-class Unsloth support |
| QLoRA over full fine-tune | Free GPU tier (Kaggle T4) — 4-bit quantization makes it feasible |
| Kaggle over Colab | 30h/week free GPU quota vs Colab's unstable free tier |
| Ollama on athena over VPS | Already running, ARM64 GGUF support, zero extra infra cost |
| MLflow for tracking | Directly closes the MLOps gap DataSpark cares about |
| GGUF format | Only format Ollama accepts for custom models |

---

## 8. Secrets & Credentials

Never hardcode. All secrets via `.env`:

```env
TELEGRAM_BOT_TOKEN=<from BotFather>
MLFLOW_TRACKING_URI=http://localhost:5000
OLLAMA_HOST=http://localhost:11434
OLLAMA_MODEL=orlon-bot:latest
```

`.env` is gitignored. See `bot/.env.example` for the template.
