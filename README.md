# orlon-bot

![CI](https://github.com/vkkatariya/orlon-bot/actions/workflows/ci.yml/badge.svg)

> End-to-end ML pipeline: personal style-tuned LLM fine-tuned on chat data, deployed as a Telegram bot.

---

## What this is

A full ML pipeline project demonstrating:

- **Data preprocessing** — Discord chat export → cleaned JSONL training pairs
- **QLoRA fine-tuning** — Llama 3.2 3B Instruct fine-tuned with Unsloth on free Kaggle GPU
- **Experiment tracking** — MLflow logging all training runs (hyperparams, loss curves, sample outputs)
- **Edge deployment** — GGUF export → Ollama on ARM64 (Radxa Rock 5T)
- **Bot integration** — Telegram bot (python-telegram-bot) with draft and auto-reply modes

---

## Architecture

```
Discord .txt exports
        │
        ▼
discord_parser.py
(burst-merge, PII redact, JSONL pairs)
        │
        ▼
Kaggle T4 GPU
Unsloth + QLoRA
Llama 3.2 3B Instruct
        │
        ▼
LoRA adapter → GGUF merge
        │
        ▼
Ollama on athena (ARM64, 24 GB)     ←── MLflow tracking
        │
        ▼
python-telegram-bot
(draft mode + safety filter)
```

---

## Pipeline Stages

| # | Stage | Tool | Status |
|---|---|---|---|
| 0 | Data pipeline | `discord_parser.py` | ✅ Done |
| 1 | Fine-tuning | Unsloth + QLoRA (Kaggle) | ⬜ |
| 2 | GGUF export | llama.cpp + Ollama | ⬜ |
| 3 | Experiment tracking | MLflow | ⬜ |
| 4 | Telegram bot | python-telegram-bot | ⬜ |
| 5 | Evaluation | BLEU + human eval | ⬜ |

---

## Quickstart

### 1. Parse your chat data
```bash
cd data/
python discord_parser.py \
  --input raw/chat1.txt raw/chat2.txt \
  --my-username your_discord_handle \
  --output processed/train.jsonl \
  --min-words 5 \
  --context-turns 3
```

### 2. Fine-tune (Kaggle)
Open `training/kaggle_notebook.ipynb` in Kaggle, enable GPU accelerator (T4), and run all cells.

### 3. Deploy on Ollama
```bash
# After downloading the GGUF from Kaggle output:
ollama create orlon-bot -f deployment/Modelfile
ollama run orlon-bot "Hey, how are you?"
```

### 4. Run the Telegram bot
```bash
cd bot/
cp .env.example .env   # fill in your tokens
uv run python main.py
```

---

## Project Structure

```
orlon-bot/
├── data/
│   ├── discord_parser.py     # Discord export → JSONL
│   ├── raw/                  # drop .txt exports here (gitignored)
│   └── processed/            # train.jsonl (gitignored)
├── training/
│   ├── kaggle_notebook.ipynb
│   └── configs/train_config.yaml
├── deployment/
│   ├── docker-compose.yml    # MLflow
│   └── Modelfile
├── bot/
│   ├── main.py
│   ├── requirements.txt
│   └── .env.example
├── eval/
│   └── eval.py
└── tasks/
    ├── DEVLOG.md
    ├── todo.md
    └── lessons.md
```

---

## Stack

| Layer | Tool |
|---|---|
| Fine-tuning | [Unsloth](https://github.com/unslothai/unsloth) + QLoRA |
| Base model | `unsloth/Llama-3.2-3B-Instruct` |
| Inference | [Ollama](https://ollama.ai) |
| Tracking | [MLflow](https://mlflow.org) |
| Bot | [python-telegram-bot](https://python-telegram-bot.org) v21+ |
| Deployment target | Radxa Rock 5T (ARM64, 24 GB RAM) |

---

## Notes

- Raw chat exports and processed JSONL are **gitignored** — they contain personal data
- The bot runs in **draft mode by default** — it suggests replies, doesn't auto-send
- All services run on a private Tailscale network — no public exposure

---

*Part of [vishalkatariya.dev](https://vishalkatariya.dev) · Built for the Orlon project*
