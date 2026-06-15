# DEVLOG.md — orlon-bot

> Append-only. Newest entry at the bottom.
> Every session ends with an entry here. No exceptions (see AGENTS.md).

---

## 2026-06-07 — Stage 0: Project Setup + Data Pipeline

### What happened
- Analyzed previous conversations (Claude DataSpark session April 2026 + ChatGPT Discord bot session) to understand the project history and unresolved decisions
- Resolved all platform/stack ambiguity: Telegram (not WhatsApp/Discord self-bot), Llama 3.2 3B, Unsloth/QLoRA on Kaggle, Ollama on athena
- Inspected Discord chat export: 143,129 messages, Sep 2024 – Jan 2026, English only, one DM conversation (ordinary_835 ↔ yellowcat0392_70924)
- Wrote `discord_parser.py` — handles burst-merge (3-min window), noise stripping (stickers/embeds/reactions/bots), PII redaction, JSONL generation with sliding context window
- Ran parser on the single export: 22,708 usable pairs, capped to 20,000 for training
- Generated and validated `train.jsonl` (15 MB, ✅ clean)
- Created project dev setup: `AGENTS.md`, `CONTEXT.md`, `README.md`, `DEVLOG.md`, `todo.md`, `lessons.md`

### What works
- `discord_parser.py` — fully functional, tested, validated
- `data/processed/train.jsonl` — 20,000 examples, clean JSONL, correct chat format
- Project structure scaffolded

### What's broken / blocked
- Training data is from ONE conversation only — model will be style-tuned to that specific dynamic. Should add 2–3 more Discord DM exports before training for better generalization
- Kaggle notebook not written yet
- GitHub repo not created yet (`vkkatariya/orlon-bot`)
- MLflow Docker Compose not set up on athena yet

### Next session starts here
1. Create GitHub repo `vkkatariya/orlon-bot`, push initial scaffold (without `data/raw/`, `data/processed/`)
2. Write `training/kaggle_notebook.ipynb` — Unsloth Llama 3.2 3B QLoRA fine-tune with MLflow logging
3. Optionally: collect 2–3 more Discord exports → re-run parser → merge into train.jsonl before training
