---
icon: material/clipboard-check-outline
---

# Lab 1 — Score, don't vibe

!!! warning "Draft"

    This lab is still being written. The starter code isn't published yet, and steps may change.

In Lab 0 you judged the agent by reading its answers and deciding whether they looked right. That works for six questions. It doesn't work when a prompt has to handle thousands of invoices, or when you change the prompt and need to know whether you made it better or worse. By the end of this lab you will have:

- [ ] labelled a 20-case test set for a real accounts payable decision, before touching the prompt
- [ ] found out where you and a teammate disagree about the "right" answer
- [ ] called a model from code and made it return data your program can check, not prose
- [ ] improved a prompt one change at a time, with a score after every change
- [ ] run the same test set on a much bigger model and compared score, speed and cost

The task comes from **Summit Ridge Outfitters** again. Every invoice that arrives has to be triaged: pay it, put it on hold, or escalate it to a person. Today a model makes that call, and you find out how often it's right.

!!! abstract "At a glance"

    | When | What | Time |
    |---|---|---|
    | **Before class** | [Part 0 — Set up](#part-0-set-up). No new downloads if Lab 0 works on your laptop. | 10 min |
    | **In class** | Parts [1](#part-1-the-task-and-a-first-look) to [5](#part-5-same-test-bigger-model) | 75 min |
    | **Submit** | Push `observations.md`, `tests/my_tests.csv`, `prompt.md` and `runs.csv` ([Part 6](#part-6-commit-and-push)). Your last push before the deadline counts. | 2 min |

!!! tip "How to use this page"

    Same as Lab 0. Every step starts with the short version. If you need more, open the **:material-chevron-right: fold** under the step.

---

## Part 0 — Set up

*Before class, about 10 minutes.*

You already have `uv`, Ollama and `qwen3:1.7b` from Lab 0. Clone this lab's repo, replacing `YOUR-GITHUB-USERNAME`, and build the environment:

```bash
git clone https://github.com/BYU-IS-456/lab-1-evals-YOUR-GITHUB-USERNAME.git
cd lab-1-evals-YOUR-GITHUB-USERNAME
uv sync
```

Then create your `.env` and check everything:

=== "macOS"

    ```bash
    cp .env.example .env
    uv run check_setup.py
    ```

=== "Windows"

    ```powershell
    copy .env.example .env
    uv run check_setup.py
    ```

=== "Linux"

    ```bash
    cp .env.example .env
    uv run check_setup.py
    ```

```ini title=".env"
MODEL=qwen3:1.7b
OLLAMA_BASE_URL=http://localhost:11434
NUM_CTX=8192
TEMPERATURE=0 # (1)!
```

1.  You saw in Lab 0 that `0` makes answers more repeatable, though not always identical. We start there so that a score change comes from your prompt and not from luck.

There are no secrets in this lab, so there's no token to paste. The habit is the same anyway: `.env` holds your settings and is never committed.

??? info "Details: what's in the repo"

    | File | What it is |
    |---|---|
    | `cases/invoices.jsonl` | 40 invoice cases, one per line, with no answers |
    | `cases/holdout.jsonl` | 20 more cases with answers. **Don't open it** until Part 4 tells you to. |
    | `data/policy.md` | The same purchasing policy as Lab 0 |
    | `prompt.md` | The system prompt. This is the file you'll improve. |
    | `triage.py` | Sends one case to the model and gets a structured decision back |
    | `score.py` | Runs every case in your test set and scores the results |
    | `tests/my_tests.csv` | Your test set. It's empty. |
    | `runs.csv` | One row per scoring run, added automatically |
    | `observations.md` | Your answers |

---

## Part 1 — The task and a first look

*About 10 minutes.*

### 1a. The decision

For every invoice, accounts payable makes one of three decisions, for one reason:

| Decision | Reason code | When |
|---|---|---|
| `pay` | `ok` | Everything checks out |
| `hold` | `no_po` | Over $2,500, no purchase order, and not a recurring contract charge |
| `hold` | `not_received` | No receiving record, or less received than invoiced |
| `hold` | `over_tolerance` | Over the PO amount by more than the allowed variance |
| `hold` | `duplicate` | A likely duplicate of an invoice already received |
| `hold` | `vendor_pending` | The vendor hasn't been approved yet |
| `escalate` | `vendor_inactive` | The vendor is inactive, so the AP manager must confirm the order was real |
| `escalate` | `bank_change` | The invoice asks us to pay a different bank account |

A `hold` waits for a document or a fix. An `escalate` needs a named person to make a judgment call. The rules behind each row are in `data/policy.md`. Skim it again now; you'll need it in Part 2.

### 1b. One case

Open `cases/invoices.jsonl` and read the first few lines. Each case is everything the AP specialist would have in front of them:

```json title="One case, spread out so you can read it"
{
  "id": "C-07",
  "invoice": {"vendor_invoice_number": "BC-4471", "amount": 5480.00, "date": "2027-01-11",
              "note": ""},
  "vendor": {"name": "Basecamp Canvas Co.", "status": "active"},
  "po": {"number": "PO-3312", "amount": 5200.00},
  "received": {"amount": 5200.00},
  "recent_invoices": []
}
```

Now let the model try three of them:

```bash
uv run triage.py C-01 C-07 C-19
```

!!! question "Answer in `observations.md`"

    1. Read the three decisions and the reasons the model gave. Do they look right? How sure are you, and what would you have to do to be sure?
    2. Work out C-07 yourself from the policy. What's the right decision? Did the model get it? (Hint: the tolerance is 5% of the PO **or** $500, whichever is smaller.)

!!! note

    The reasons the model writes are fluent and confident whether or not the decision is right. Reading them tells you very little. That's the problem this lab solves.

---

## Part 2 — Write the test set first

*About 20 minutes.*

Before you change the prompt, decide what "right" means. A **test set** is a list of cases with the answer you expect for each one. You write it first so that the model's answers can't talk you into anything.

### 2a. Label 20 cases

Pick 20 cases from `cases/invoices.jsonl` and record your answer for each in `tests/my_tests.csv`. Decide from the policy, by hand. Don't ask a model.

```csv title="tests/my_tests.csv"
id,expected_decision,expected_reason,why
C-07,hold,over_tolerance,"$280 over the PO; the limit is $260 (5% of $5,200)"
```

Your 20 must include:

- at least five `pay` cases,
- every reason code in the table at least once,
- at least three cases you found hard to decide.

Then check your file:

```bash
uv run score.py --check
```

??? info "Details: why not just use all 40, or let AI label them?"

    Labelling is the work. Every case you label by hand forces you to apply the policy yourself, and that's how you find out where the policy is vague. A test set labelled by a model measures whether your model agrees with that model, which isn't what you want to know.

    Twenty is small enough to finish in class and large enough that one case is worth 5 points. Professional test sets run from dozens of cases to thousands, and they grow every time something goes wrong in production.

??? info "Details: what makes a good test case"

    - **Easy cases** tell you the prompt isn't broken. You need some, but they all pass or fail together.
    - **Boundary cases** sit right at a rule's edge: $2,500 exactly, a variance of exactly 5%, an invoice number that differs by one hyphen.
    - **Conflict cases** trigger two rules at once: an inactive vendor *and* no PO. Which reason wins?
    - **Trap cases** look like a problem but aren't: the same rent amount every month is not a duplicate.

    The `why` column is for you. In a month you won't remember why C-23 was a hold.

### 2b. Trade with a teammate

Pick five of your cases, the harder the better. Give a teammate only the case IDs, not your answers. They label your five; you label theirs. Then compare.

!!! question "Answer in `observations.md`"

    3. On how many of the ten cases did the two of you disagree? Pick one. Who was right, or is the policy unclear?
    4. For a case where the policy is unclear, what should happen to it: fix the label, fix the policy, or drop the case? Why?
    5. If two people who read the policy disagree on some cases, what's the highest score you should expect from a model? What does a score of 100% mean?

Fix any labels you changed your mind about before you go on. From here on, **don't edit your test set to make the score go up.**

---

## Part 3 — Get an answer your code can check

*About 10 minutes.*

### 3a. Read the call

Open `triage.py`. It's about 40 lines. **Read it before you run anything else.**

```python title="triage.py (the part that matters)"
class Triage(BaseModel):
    decision: Literal["pay", "hold", "escalate"]
    reason_code: Literal["ok", "no_po", "not_received", "over_tolerance",
                         "duplicate", "vendor_pending", "vendor_inactive", "bank_change"]
    explanation: str


response = client.chat(
    model=config.MODEL,
    messages=[
        {"role": "system", "content": SYSTEM_PROMPT},      # (1)!
        {"role": "user", "content": json.dumps(case)},     # (2)!
    ],
    format=Triage.model_json_schema(),                     # (3)!
    options={"temperature": config.TEMPERATURE, "num_ctx": config.NUM_CTX},
)
result = Triage.model_validate_json(response.message.content)  # (4)!
```

1.  The contents of `prompt.md`. The same for every case.
2.  One invoice case, as JSON. Different every time.
3.  Tells Ollama the reply must be JSON in exactly this shape.
4.  Checks that the reply really does match, and turns it into a Python object.

??? info "Details: what structured output is and why it matters"

    In Lab 0 the model answered in sentences, and you read them. A program can't do that. If the next step in the process is "release the payment" or "open a ticket for the AP manager", your code needs a value it can test with `if`, not a paragraph.

    **Structured output** means the model is constrained to reply in a format you define, here a JSON object with three fields. `decision` and `reason_code` can only be one of the listed values. The model can't answer "probably pay, but check with someone".

    The schema is a guarantee about *shape*, not about *truth*. The model can return perfectly valid JSON that says `pay` for an invoice that should be held. That's what the test set is for.

    **Validation** is your code refusing to trust the reply until it has checked it. Always validate, even when the model is told to follow a schema.

### 3b. Get a baseline

```bash
uv run score.py --note "baseline"
```

This runs all 20 of your cases, compares each result to your label, prints the misses, and adds a row to `runs.csv`.

```text title="What the output looks like"
C-07  expected hold/over_tolerance   got pay/ok            MISS
C-12  expected hold/duplicate        got hold/duplicate    ok
...
decision 12/20 (60%)   decision+reason 9/20 (45%)   38 s   21,400 tokens in   1,150 out
```

!!! question "Answer in `observations.md`"

    6. What's your baseline score for the decision, and for decision plus reason? Why track both?
    7. Look at the misses. Are they spread evenly, or is the model bad at one kind of case? Which kind?
    8. If the model answered `pay` for every single case, what would it score on your test set? What does that tell you about your baseline?

---

## Part 4 — Improve the prompt against the score

*About 20 minutes.*

Open `prompt.md`. Right now it's two sentences. Make it better, **one change at a time**, and score after every change:

```bash
uv run score.py --note "added the variance rule"
```

Make at least four changes. Here are some to try, roughly in the order professionals reach for them:

??? example "1. Say what the rules are"

    The model has never seen Summit Ridge's policy. Paste in the sections that matter, or summarize them as rules. Try both and compare.

??? example "2. Show it examples"

    Add two or three worked cases to the prompt, with the right answer for each. Use cases from `cases/invoices.jsonl` that are **not** in your test set. Why does that matter?

??? example "3. Do the math for it"

    In Lab 0 you saw how `$20,440.00` gets split into tokens. The model is being asked to compute 5% of a PO amount and compare numbers. Run with `--facts`:

    ```bash
    uv run score.py --facts --note "computed the variance in Python"
    ```

    This adds a few values that Python computes to each case before the model sees it: the amount over the PO, the tolerance, and whether a PO is required. Open `triage.py` and find where that happens.

??? example "4. Give it an order to check things in"

    When two rules apply, which wins? Tell the model the order to check in and to stop at the first problem.

??? example "5. Let it think"

    ```bash
    uv run score.py --think --note "thinking on"
    ```

    Watch the time and the output tokens as well as the score.

??? example "6. Run the same prompt twice"

    Change nothing and score again. Then set `TEMPERATURE=0.7` in `.env` and score twice more.

??? info "Details: this is context engineering"

    Everything the model reads is the prompt: your instructions, the policy, the examples, and the case itself. Changing the wording of an instruction is the weakest tool you have. The stronger ones change *what information is in the window*: adding the rule the model lacked, showing it a solved case, or handing it a number so that it doesn't have to compute one.

    Experiment 3 is the most important one in this lab. Models are unreliable at arithmetic and code is perfect at it, so do the arithmetic in code and let the model do the reading and judging.

### 4b. The cases you haven't seen

You've now adjusted your prompt many times against the same 20 cases. Find out whether it learned the task or just those cases:

```bash
uv run score.py --holdout --note "holdout"
```

This scores your current prompt on 20 cases you've never looked at, labelled by the instructor.

!!! question "Answer in `observations.md`"

    9. Paste your `runs.csv`. Which single change helped most? Did you expect that one?
    10. Find a change that fixed at least one case and broke at least one other. What happened? How would you have noticed without the test set?
    11. What did `--facts` do to your score? What does that suggest about dividing work between code and a model?
    12. When you ran the same prompt twice, did the score change? By how much? How big does an improvement need to be before you believe it?
    13. How did your holdout score compare with your own test set's score? Give one reason they might differ. (You've seen this idea in IS 455.)

---

## Part 5 — Same test, bigger model

*Remaining time.*

Your test set doesn't care which model answers. That makes it the tool for choosing one.

### 5a. A bigger local model

If your laptop has the memory for it (see the table in Lab 0), try the next size up:

=== "macOS"

    ```bash
    ollama pull qwen3:4b
    MODEL=qwen3:4b uv run score.py --note "qwen3:4b"
    ```

=== "Windows"

    ```powershell
    ollama pull qwen3:4b
    $env:MODEL="qwen3:4b"; uv run score.py --note "qwen3:4b"
    Remove-Item Env:MODEL
    ```

=== "Linux"

    ```bash
    ollama pull qwen3:4b
    MODEL=qwen3:4b uv run score.py --note "qwen3:4b"
    ```

### 5b. A frontier model

Use your BYU ChatGPT account. Export your prompt and all 20 cases as one message, paste it into a new chat, and save the reply:

```bash
uv run score.py --export chatgpt_prompt.txt
```

1. Paste the contents of `chatgpt_prompt.txt` into a new ChatGPT conversation.
2. Copy the JSON it returns into a new file named `chatgpt_answers.json`.
3. Score it:

```bash
uv run score.py --answers chatgpt_answers.json --note "ChatGPT"
```

??? info "Details: why paste, and what's different about it"

    A program would normally call a hosted model through its API, the same way `triage.py` calls Ollama. We're pasting by hand today because it needs no API key and costs nothing.

    It isn't quite the same test. All 20 cases go in one message, so the model sees them together, and the chat app adds its own instructions that you can't see. Keep that in mind when you compare.

!!! question "Answer in `observations.md`"

    14. Fill in the comparison table: score, time and tokens for each model you ran.
    15. Look up the current price per million input and output tokens for the hosted model you used. Using the token counts from your local run, about what would 20 cases cost? What about 5,000 invoices a month?
    16. Which cases did the bigger model still miss? Are they the same ones you and your teammate disagreed about?
    17. Summit Ridge wants to use this for real. Which model would you recommend, and what would you tell them about the invoices it gets wrong?
    18. **Last question.** In two or three sentences: why write the test set before the prompt? Use the words *score*, *change* and *holdout*.

---

## Part 6 — Commit and push

```bash
git status      # should NOT list .env
git add observations.md tests/my_tests.csv prompt.md runs.csv
git commit -m "Lab 1 complete"
git push
```

Pushing is submitting: your last push before the deadline counts. Push as often as you like.

---

## Troubleshooting

??? failure "`score.py --check` says a case ID doesn't exist"

    IDs look like `C-07`, with a capital C, a hyphen and two digits. Check for a trailing space, or for a spreadsheet app that changed the file. Edit the CSV in your code editor, not in Excel.

??? failure "`score.py --check` says a reason code is missing"

    Your 20 cases must cover all eight reason codes. The message lists the ones you don't have yet. Look through `cases/invoices.jsonl` for a case that fits.

??? failure "`ValidationError` when a case runs"

    The model returned something that doesn't match the schema. It's rare with `format=` set, but it happens with very small models or a context window that's too small. `score.py` counts it as a miss and keeps going. If you see many, check that `NUM_CTX` is at least `8192`.

??? failure "A scoring run takes more than two minutes"

    Close other apps. Turn off `--think` if you left it on. The first case is slow while the model loads.

??? failure "Your prompt got long and the score dropped"

    A long prompt plus the case may no longer fit in the context window, and Ollama cuts the start of it without telling you. You saw this in Lab 0. Shorten the prompt or raise `NUM_CTX`.

??? failure "ChatGPT's reply won't score"

    `chatgpt_answers.json` must contain only the JSON list, with no sentence before or after it and no code-fence lines. If ChatGPT skipped cases, ask it for the ones that are missing.

??? failure "Ollama isn't running, or the model isn't found"

    See [Lab 0's troubleshooting](lab0.md#troubleshooting).
