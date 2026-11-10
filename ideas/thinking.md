# Thinking: what enables it, what it enables

Status: idea. Not for Lab 0.

## Questions to build around

1. What about model design or harness design enables thinking?
2. What does thinking enable?

## Hooks from Lab 0

- `step2_agent_loop.py` asks Ollama to return thinking separately (`ChatOllama(reasoning=True)`) and prints it as `[thinking]`. Students watch it for a whole class without it being explained.
- `step1_chat.py` turns thinking off with `think: False`.
- `qwen3:1.7b` is a hybrid model that can switch thinking on and off, so running the same question both ways costs nothing.

## Rough outline

### What enables thinking

- **Model design:**
  - **Training.** Fine-tuning and reinforcement learning on problems with checkable answers reward the model for writing out reasoning before it answers.
  - **The chat template.** It gives that reasoning its own channel (`<think>` tokens), separate from the answer.
- **Harness design:** the code around the model decides:
  - whether thinking is requested at all (`think` / `reasoning=True`)
  - how many tokens it may spend (a thinking budget)
  - whether thinking is shown to the user, hidden, or summarized
  - whether earlier thinking is sent back to the model on later steps, or dropped to save context

### What thinking enables

- **More computation per answer.** The model spends extra tokens working a problem out before it answers.
- **A chance to correct itself.** It can catch a mistake, plan tool calls, and break a problem into steps.
- **The costs, which students can measure with Lab 0's token counts:**
  - thinking tokens are output tokens you pay for
  - answers take longer
  - thinking fills up the context window
- **A caveat for the discussion.** The written-out thinking isn't guaranteed to be the model's real reasoning. Students should read it as evidence, not a faithful record.
