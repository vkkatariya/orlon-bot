# lessons.md — orlon-bot

> Hard-won lessons. Add to this file whenever something bites you.

---

## Data & Parsing

### Lesson 1 — Discord burst messages need merging
**Date:** 2026-06-07
**Symptom:** Training pairs looked like: `[USER] Yeah` → `[ASSISTANT] i know right`
**Cause:** Discord users send multiple short messages in rapid succession. Each message was treated as a separate training example, producing near-useless 1-2 word responses.
**Fix:** Merge consecutive same-speaker messages within a 3-minute window before building pairs.
**Key principle:** Always merge bursts before any pair-building step.

---

### Lesson 2 — 46% of raw messages are ≤3 words
**Date:** 2026-06-07
**Context:** After parsing 66k Vishal messages, nearly half were 1-3 words ("Yeah", "Ohh", "Haha", "Yesss").
**Impact:** These add noise and teach the model to be unresponsive. The `--min-words 5` filter on responses is essential.
**Key principle:** Filter short responses, not short contexts. A short context + meaningful response is still a good training example.

---

### Lesson 3 — Single-conversation data creates person-specific overfitting risk
**Date:** 2026-06-07
**Context:** All 20,000 training pairs come from one Discord DM conversation (Miramuffins). The model may learn to respond specifically to that person's style/topics.
**Fix:** Add 2–3 more Discord exports from different conversations before training.
**Key principle:** Training data diversity = generalization. More people = more robust style model.

---

### Lesson 4 — Discord self-bots are against Discord ToS
**Date:** 2026-06-07 (from ChatGPT conversation analysis)
**Context:** Early in brainstorming, the idea of automating replies from a personal Discord account was considered.
**Fix:** Use Telegram Bot API instead. It's official, well-documented, and designed for this.
**Key principle:** Never automate a personal user account on any platform. Always use official bot APIs.

---

## Training

*(Add entries here as training runs complete)*

---

## Deployment

*(Add entries here as deployment issues arise)*

---

## Telegram Bot

*(Add entries here as bot issues arise)*
