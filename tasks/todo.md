# todo.md — orlon-bot

> Organized by pipeline stage. Work top to bottom.

---

## 🔄 Stage 0 — Data (in progress)

- [x] Write `discord_parser.py`
- [x] Generate `train.jsonl` (20,000 pairs, 1 export)
- [ ] Add 2–3 more Discord DM exports to `data/raw/`
- [ ] Re-run parser with all exports merged → new `train.jsonl`
- [ ] Create GitHub repo `vkkatariya/orlon-bot`
- [ ] Push initial scaffold (gitignore raw + processed data)
- [ ] Add `.gitignore` (data/raw/, data/processed/, *.gguf, .env, __pycache__)

---

## ⬜ Stage 1 — Fine-Tuning (Kaggle)

- [ ] Write `training/kaggle_notebook.ipynb`
  - [ ] Unsloth Llama 3.2 3B Instruct load
  - [ ] QLoRA config (r=16, alpha=16, 4-bit)
  - [ ] Upload `train.jsonl` to Kaggle dataset
  - [ ] Training loop (2–3 epochs)
  - [ ] MLflow logging (params + loss per step)
  - [ ] Save LoRA adapter as Kaggle output
- [ ] Write `training/configs/train_config.yaml`
- [ ] Run notebook on Kaggle T4 GPU
- [ ] Download trained adapter

---

## ⬜ Stage 2 — GGUF Export + Ollama

- [ ] Merge LoRA adapter → full model weights
- [ ] Convert to GGUF (Q4_K_M quantization)
- [ ] Write `deployment/Modelfile`
- [ ] Pull GGUF to athena via scp/Mutagen
- [ ] `ollama create orlon-bot -f deployment/Modelfile`
- [ ] Test: `ollama run orlon-bot "Hey how are you?"`
- [ ] Verify response quality vs base Llama 3.2 3B

---

## ⬜ Stage 3 — MLflow on athena

- [ ] Write `deployment/docker-compose.yml` for MLflow
- [ ] Deploy at `/srv/dev-shared/homelab/mlflow/`
- [ ] Confirm MLflow UI accessible at `http://127.0.0.1:5000`
- [ ] Optionally expose via Caddy + tailscale cert on athena
- [ ] Log Stage 1 training run retroactively if needed

---

## ⬜ Stage 4 — Telegram Bot

- [ ] Create Telegram bot via BotFather → get token
- [ ] Write `bot/main.py`
  - [ ] Message handler with safety filter (risky keywords → draft only)
  - [ ] Ollama API call → generate reply
  - [ ] Draft mode: bot DMs reply suggestion to you
  - [ ] `.env` config loading
- [ ] Write `bot/requirements.txt`
- [ ] Write `bot/.env.example`
- [ ] Deploy on athena via pm2: `pm2 start bot/main.py --interpreter python3`
- [ ] `pm2 save` + `pm2 startup` for persistence across reboots

---

## ⬜ Stage 5 — Evaluation

- [ ] Write `eval/eval.py`
  - [ ] Load 100 held-out pairs from `data/processed/val.jsonl`
  - [ ] Run both base Llama 3.2 3B and fine-tuned model
  - [ ] Compute BLEU score comparison
  - [ ] Output side-by-side sample table
- [ ] Add val split to parser (90/10 split flag)
- [ ] Document results in README

---

## ⬜ Stage 6 — Polish + GitHub

- [ ] Update `README.md` with actual results (BLEU, sample outputs)
- [ ] Add architecture diagram to README
- [ ] Push all code to `vkkatariya/orlon-bot`
- [ ] Write Notion Blueprint entry for orlon-bot
- [ ] Add project to portfolio (vishalkatariya.dev)
