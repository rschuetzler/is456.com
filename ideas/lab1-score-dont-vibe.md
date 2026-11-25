# Lab 1 — Score, don't vibe

Status: outline, from the v0.3 schedule.

| | |
|---|---|
| When | Week 3, Thu Jan 21 |
| Who | Individual |
| Tools | Local model, then ChatGPT |

## What students do

- Task: triage Summit Ridge invoices into pay, hold or escalate, with a reason, as structured output.
- Write a 20-case test set with expected answers first. Teammates trade five cases to catch ambiguous ones.
- Iterate the system prompt against the test set on the local model, recording the score after every change.
- Run the final prompt on a frontier model and compare score, speed and cost.

## What they should notice

- A change that fixes one case can break two others.
- Ambiguous test cases are a finding about the task.
- A bigger model hides failures the small one showed.

## To build

Starter repo with invoice cases, a scoring script and a structured-output call.
