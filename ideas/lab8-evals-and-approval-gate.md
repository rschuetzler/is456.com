# Lab 8 — Evals and an approval gate

Status: outline, from the v0.3 schedule.

| | |
|---|---|
| When | Week 11, Thu Mar 18 |
| Who | Team; evals lead. Check-off 2 |
| Tools | Local or API model |

## What students do

- Add a human approval step before the agent's irreversible action.
- Trace runs and grade the full suite, with a model judge for cases a rule cannot grade.
- Check the judge against ten hand labels.
- Report score and cost per case.
- Check-off 2: any member explains any layer and changes it, with a TA and no AI.

## What they should notice

- The judge disagrees with you on some cases and is sometimes right.
- An approval prompt with too little information gets approved every time.
- Cost per case is a design input.

## To build

Tracing setup; a judge prompt template.
