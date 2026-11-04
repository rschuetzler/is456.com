---
icon: material/robot-outline
---

# Lab 0 — Your first agent

You have used AI assistants plenty. Today you'll look inside one. By the end of this lab you will have:

- [ ] a Python project managed with `uv`, with its secrets kept in a `.env` file that git never sees
- [ ] a small language model running on your own laptop with Ollama
- [ ] seen exactly how text becomes the tokens a model reads and pays for
- [ ] read every line of a working agent loop and watched it reason, call tools, make mistakes and recover
- [ ] changed the agent by giving it a new tool, and seen how that changes its behavior

The agent works for the accounts payable team at **Summit Ridge Outfitters**, the company whose procure-to-pay process we'll use all semester. It can query the company's purchasing database and read its purchasing policy, and it can't change anything.

!!! abstract "At a glance"

    | When | What | Time |
    |---|---|---|
    | **Before class** | [Part 0 — Set up](#part-0-set-up). It downloads about 1.5 GB, and 70 people doing that on campus Wi-Fi at once won't work. | 30 min |
    | **In class** | Parts [1](#part-1-what-the-model-actually-sees) to [5](#part-5-experiments) | 75 min |
    | **Submit** | Push `observations.md` and your updated `tools.py` ([Part 6](#part-6-commit-and-push)) | 2 min |

!!! tip "How to use this page"

    Every step starts with the short version. If you already know how, just do it. If not, open the **:material-chevron-right: fold** under the step for the full walkthrough.

---

## Part 0 — Set up

*Before class, about 30 minutes.*

### 0.0 Install mise (recommended)

[mise](https://mise.jdx.dev/) installs and updates developer tools with the same command on every OS. The install steps below show the **mise** command first. If you'd rather not use mise, skip to [0.1](#01-install-uv) and pick your OS's tab instead. Either way, pick your OS's tab in steps that have no mise tab.

=== "macOS"

    ```bash
    curl -fsSL https://mise.run/zsh | sh
    ```

=== "Windows"

    ```powershell
    $env:MISE_INSTALL_SHIMS_PATH=1; irm https://mise.run/powershell | iex
    ```

=== "Linux"

    ```bash
    curl -fsSL https://mise.run/bash | sh
    ```

Close and reopen your terminal, then check with `mise --version`.

??? info "Details: what mise is, and what that command set up"

    Every tool in this course (`uv`, Ollama, and later ones) has its own installer, and each OS installs it differently. mise replaces all of that with one command: `mise use -g <tool>` installs the latest version of a tool, and `mise upgrade` updates everything you've installed. The `-g` means global: the tool works in every folder, not just this project.

    The command does two things. It installs `mise` into `~/.local/bin`, and it makes your terminal find the tools mise installs. On macOS and Linux it adds a line to `~/.zshrc` or `~/.bashrc`. On Windows it adds mise's `shims` folder to your `Path`.

    The macOS command assumes zsh and the Linux one assumes bash, the default shell on each. If you use a different shell, see [mise's install guide](https://mise.jdx.dev/installing-mise.html).

### 0.1 Install uv

=== "mise"

    ```bash
    mise use -g uv
    ```

=== "macOS"

    ```bash
    brew install uv
    ```

    ??? info "No Homebrew?"
        
        I recommend Homebrew ([https://brew.sh/](https://brew.sh/)) for installing Mac developer tools. But if you don't trust me (or it), you can use the standalone installer instead. It puts `uv` in `~/.local/bin` and adds that folder to your `PATH`.

        ```bash
        curl -LsSf https://astral.sh/uv/install.sh | sh
        ```


=== "Windows"

    ```powershell
    powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
    ```

=== "Linux"

    ```bash
    curl -LsSf https://astral.sh/uv/install.sh | sh
    ```

Close and reopen your terminal, then check with `uv --version`.

??? info "What is uv?"

    `uv` is a fast Python project manager. It installs the right version of Python for you, creates a virtual environment, and installs packages, all from one tool.

### 0.2 Install Ollama and download a model

=== "mise"

    ```bash
    mise use -g ollama
    ollama serve    # leave this running in its own terminal window
    ```

    mise installs the `ollama` command-line program, but not the desktop app or background service, so nothing starts Ollama for you. Whenever you work on a lab, start `ollama serve` in its own terminal window and leave it open. Run everything else in a second window. If you'd rather Ollama start by itself, use your OS's tab instead.

=== "macOS"

    Install Ollama from **<https://ollama.com/download>** and start the app.

=== "Windows"

    Install Ollama from **<https://ollama.com/download>** and start the app.

=== "Linux"

    ```bash
    curl -fsSL https://ollama.com/install.sh | sh
    ```

    It asks for your password, then sets Ollama up as a background service that starts on its own. If it says it needs `zstd`, install that first (`sudo apt install zstd` on Ubuntu or Debian, `sudo dnf install zstd` on Fedora) and run it again.

Then download the model and try it:

```bash
ollama pull qwen3:1.7b
ollama run qwen3:1.7b   # ask it anything, then type /bye
```

??? info "Details: what Ollama is and why such a small model"

    Ollama runs language models locally. Nothing you send it leaves your laptop, and it costs nothing per call. On macOS and Windows it runs in the menu bar or system tray once you start the app. On Linux it runs as a `systemd` service named `ollama`, with no window or icon.

    `qwen3:1.7b` is a 1.7-billion-parameter model, about 1.4 GB on disk and under 3 GB of memory while it runs. For comparison, the models behind commercial chat assistants are estimated at hundreds of billions to trillions of parameters. Expect it to be noticeably less capable. That's useful: you'll see failures today that a bigger model would hide.

    When you run it, you'll see it "think" first before it answers.

### 0.3 Get the starter code and build the environment

```bash
git clone <your GitHub Classroom repo URL>
cd <the repo folder>
uv sync # (1)!
```

1.  Reads `.python-version`, `pyproject.toml` and `uv.lock`, then installs Python 3.12 and every package into `.venv/`. Open the fold below for what each file does.

??? info "Details: what `uv sync` just did, and how to run code"

    `uv sync` read three files and did three things:

    | File | What it says | What `uv sync` did |
    |---|---|---|
    | `.python-version` | this project uses Python 3.12 | downloaded Python 3.12 if you didn't have it |
    | `pyproject.toml` | the packages this project needs (`langchain`, `langchain-ollama`, …) | created a virtual environment in `.venv/` and installed them there |
    | `uv.lock` | the exact version of every package, including dependencies of dependencies | installed exactly those versions, so your environment matches everyone else's |

    A **virtual environment** is a private copy of Python and packages for one project, so projects can't break each other. You run code inside it with `uv run`:

    ```bash
    uv run step2_agent_loop.py   # this project's Python and packages
    ```

    You never need to "activate" anything. If you want to, it's `source .venv/bin/activate` on macOS or Linux, or `.venv\Scripts\activate` on Windows. Later in the semester you'll add packages with `uv add <package>`, which updates `pyproject.toml` and `uv.lock` together.

### 0.4 Set up your `.env` file

1. Create a **Read** token at <https://huggingface.co/settings/tokens> named `is456`.
2. Copy the template and paste your token after `HF_TOKEN=`.
3. Confirm git ignores `.env`.

=== "macOS"

    ```bash
    cp .env.example .env
    git check-ignore -v .env    # prints the rule that hides it
    ```

=== "Windows"

    ```powershell
    copy .env.example .env
    git check-ignore -v .env    # prints the rule that hides it
    ```

=== "Linux"

    ```bash
    cp .env.example .env
    git check-ignore -v .env    # prints the rule that hides it
    ```

```ini title=".env"
MODEL=qwen3:1.7b
OLLAMA_BASE_URL=http://localhost:11434
NUM_CTX=8192 # (1)!
HF_TOKEN=hf_yourTokenHere # (2)!
HF_TOKENIZER=Qwen/Qwen3-1.7B
```

1.  Context window in tokens. Bigger remembers more of a long conversation but uses more memory. 8192 keeps `qwen3:1.7b` under about 3 GB.
2.  No spaces and no quotes. Hugging Face shows the token only once; if you lose it, make a new one.

!!! danger "Rules for secrets, this semester and after"

    Never put a secret in code, a commit, a screenshot, Slack, or a prompt to an AI assistant (including a coding agent that can read your files). If you think one has leaked, don't just delete it: **revoke it** at the provider and make a new one.

??? info "Details: why environment variables, and why a `.env` file"

    Programs need configuration (which model? which server?) and secrets (passwords, API keys). Neither belongs in your code:

    - **Configuration changes between machines.** Your laptop might need a smaller model than your teammate's. If the model name is in the code, you'd fight over it in every commit.
    - **Secrets in code end up in git, and git history is forever.** Bots scan GitHub for keys around the clock. A key pushed to a public repo is typically found and abused within minutes, and deleting it in a later commit doesn't remove it from the history.

    The standard answer is **environment variables**: named values that live in the environment a program runs in, not in its code. Python reads them with `os.getenv("NAME")`. Look at `config.py` to see every variable this lab uses.

    You could set them in your terminal before every run, but that gets tedious, so projects keep them in a file named **`.env`** and load it at startup (that's the `load_dotenv()` line in `config.py`).

    Notice the pair: **`.env.example`** is committed and tells teammates which variables exist. **`.env`** holds your real values and is never committed. You'll use this pattern in every project from now on.

??? info "Details: getting the Hugging Face token, step by step"

    Hugging Face is where most open models, including ours, are published. You'll use it in Part 1 to download your model's tokenizer.

    1. Create a free account at <https://huggingface.co> (or sign in).
    2. Go to **Settings → Access Tokens** and click **Create new token**.
    3. Choose token type **Read**, name it `is456`, and create it.
    4. Copy the token (it starts with `hf_`).

    Then open `.env` in your editor and paste it after `HF_TOKEN=`. Finally, run `git status`: `.env` must **not** appear in the list.

??? example "Try it: which value wins?"

    A variable set in your terminal overrides the same variable in `.env`. That's handy for trying something once without editing the file.

    === "macOS"

        ```bash
        MODEL=some-other-model uv run python -c "import config; print(config.MODEL)"
        uv run python -c "import config; print(config.MODEL)"
        ```

    === "Windows"

        ```powershell
        $env:MODEL="some-other-model"; uv run python -c "import config; print(config.MODEL)"
        Remove-Item Env:MODEL; uv run python -c "import config; print(config.MODEL)"
        ```

    === "Linux"

        ```bash
        MODEL=some-other-model uv run python -c "import config; print(config.MODEL)"
        uv run python -c "import config; print(config.MODEL)"
        ```

    The first command prints `some-other-model`; the second prints `qwen3:1.7b` from your `.env`.

### 0.5 Check everything

```bash
uv run check_setup.py
```

Fix the first `FAIL` it reports and run it again until every line passes. The first model call can take 30 seconds while the model loads. Stuck? See [Troubleshooting](#troubleshooting), then bring your laptop to class early.

---

## Part 1 — What the model actually sees

*About 10 minutes.*

### 1a. Tokens

```bash
uv run step1_tokens.py
uv run step1_tokens.py "your text here"
```

??? info "What a tokenizer is"

    This downloads the tokenizer for your model from Hugging Face, using your token. A tokenizer is the fixed dictionary that turns text into the numbered pieces (tokens) a model reads and writes. Models never see letters or words, only token IDs.

!!! question "Answer in `observations.md`"

    1. How is `INV-1045 for $20,440.00` split up? What does that suggest about asking a language model to do arithmetic on invoice amounts?
    2. `strawberry` is one token. Why might a model struggle to count the r's in it?
    3. The script ends with the "fixed cost" of every agent step. What is it, and where does it come from?

### 1b. One raw model call

```bash
uv run step1_chat.py                   # the same prompt, three times
uv run step1_chat.py --temperature 0   # randomness turned down
```

!!! question "Answer in `observations.md`"

    4. Did the three answers agree? Did `--temperature 0` make them identical? What does that mean for a business process that needs the same answer every time?

!!! note

    The model answered a question about a specific invoice with no access to the invoice, the PO or the policy. Keep that in mind for Part 2.

---

## Part 2 — Read the agent loop, then run it

*About 25 minutes.*

### 2a. Read before you run

Open `step2_agent_loop.py` and `tools.py`. The entire agent is the `run_agent()` function, about 40 lines. **Don't run anything yet.**

!!! question "Answer in `observations.md`"

    5. Find the three numbered comments in the loop. In one sentence each, what happens at 1, 2 and 3?
    6. Who runs the SQL query: the model or your Python code? What exactly does the model produce instead?
    7. How does the loop decide it's finished? Name both ways it can stop.
    8. What is in `messages` by the third step? Why does the whole list get sent to the model every time?
    9. In `tools.py`, where does the model get its description of each tool? What would happen if a docstring were wrong?

### 2b. Run it and watch

```bash
uv run step2_agent_loop.py "How many active vendors do we have?"
```

Then run each question below, fresh, and record what happens in the table in `observations.md`.

| # | Question | What to watch for |
|---|---|---|
| A | `How many active vendors do we have?` | The shortest path to an answer |
| B | `Which invoices are for more than their purchase order amount?` | How it discovers the schema; whether its SQL is right |
| C | `Are there any duplicate invoices?` | What it decides "duplicate" means |
| D | `Should we pay invoice INV-1045? Check our purchasing policy.` | `lookup_policy` isn't built yet. What does the agent do with the error? |
| E | `Delete the duplicate invoice.` | The database is opened read-only. What happens, and what does the agent tell you? |
| F | `What's the weather in Provo today?` | No tool can answer this |

??? info "How to read the output"

    The output labels each piece:

    - **[thinking]** is the model reasoning to itself.
    - **[tool call]** is what it asked your code to run.
    - **[tool result]** is what your code sent back.
    - The final line reports model calls, tokens and time.

    The agent has no memory between runs, so each question starts from nothing. Run one of them again with `--show-messages` to print the full conversation exactly as the model last saw it.

!!! question "Answer in `observations.md`"

    10. Pick one question where the agent got the wrong answer or wrote bad SQL. What went wrong, and did it notice?
    11. Compare the input tokens from A and B. Why does a longer task cost so much more than the number of steps alone suggests? (Hint: question 8.)
    12. In D, the agent got an error back from a tool. Did it say so honestly, work around it, or make something up?

---

## Part 3 — The same agent in LangChain

*About 10 minutes.*

```bash
uv run step3_langchain_agent.py "Which invoices are for more than their purchase order amount?"
```

`step3_langchain_agent.py` builds the same agent (same model, tools and system prompt) with LangChain's `create_agent`. Read it, then run a couple of the Part 2 questions.

!!! question "Answer in `observations.md`"

    13. Where did the loop go? Point to the line that replaced your 1-2-3 loop.
    14. Run question D or E again. LangChain formats tool errors differently from your loop. Find one difference. What other decisions might a framework be making for you that you can't see?

---

## Part 4 — Change the code: build the policy tool

*About 15 minutes.*

Implement `lookup_policy` in `tools.py`. Test it on its own, then rerun question D.

```bash
uv run python -c "from tools import lookup_policy; print(lookup_policy.invoke({'topic': 'duplicate'}))"
```

??? info "Details: what to write"

    Its body is a `TODO` that returns an error, which is what you saw in question D. `load_policy_sections()` is already written for you; your code will be about six lines. The comment in the stub gives you the steps:

    ```python title="tools.py" hl_lines="4-11"
    @tool
    def lookup_policy(topic: str) -> str:
        """Look up the company's purchasing and accounts payable policy on a topic, e.g. 'three-way match', 'duplicate invoices', 'new vendors'."""
        # TODO (Part 4): replace the line below with a real implementation.
        #
        #   1. sections = load_policy_sections()
        #   2. Split `topic` into lowercase words.
        #   3. Keep every section whose heading or text contains any of those words.
        #   4. Return the matching sections as "## heading\n\ntext", joined by blank
        #      lines. If nothing matches, return a message that says so AND lists
        #      the available section headings, so the model can try again.
        return "ERROR: lookup_policy is not implemented yet."
    ```

    When it returns policy text, rerun question D, and add one of your own that needs both the database and the policy, for example `Which pending invoices would our policy put on hold, and why?`

!!! question "Answer in `observations.md`"

    15. Paste the trace summary (model calls, tool calls, tokens) for D before and after. What changed in the agent's behavior, apart from it now having the answer?
    16. Call your tool with the topic `duplicate invoices` and count how many sections come back. Why so many? What are the limits of matching on keywords? (We fix this properly in week 4.)

---

## Part 5 — Experiments

*Remaining time. Do at least two.* Change one thing at a time, run, record what happened, and change it back.

??? example "1. Same question, different path"

    Run question B three times. Does the agent take the same steps each time? Does it get the same answer?

??? example "2. Starve it"

    Set `MAX_STEPS = 2` in `step2_agent_loop.py` and run B.

??? example "3. Sabotage a description"

    Change the docstring of `run_sql` to `"Does stuff."` and run A.

??? example "4. Weaken the instructions"

    Delete `Look things up instead of guessing.` from `SYSTEM_PROMPT` and run D or F.

??? example "5. Smaller or bigger brain"

    ```bash
    ollama pull qwen3:0.6b
    ```

    === "macOS"

        ```bash
        MODEL=qwen3:0.6b uv run step2_agent_loop.py "Which invoices are for more than their purchase order amount?"
        ```

    === "Windows"

        ```powershell
        $env:MODEL="qwen3:0.6b"; uv run step2_agent_loop.py "Which invoices are for more than their purchase order amount?"
        ```

    === "Linux"

        ```bash
        MODEL=qwen3:0.6b uv run step2_agent_loop.py "Which invoices are for more than their purchase order amount?"
        ```

    If your laptop has 8 GB or more of memory to spare, try `qwen3:4b` too. Compare steps, errors and answers.

??? example "6. Shrink its memory"

    Run B with `NUM_CTX=1024`. When the conversation outgrows the context window, Ollama quietly cuts older parts of it to make it fit, without telling your code. What breaks?

!!! question "Answer in `observations.md`"

    17. For each experiment you ran: what you changed, what you expected, what happened.
    18. **Last question.** In two or three sentences: after today, what is an agent? Use the words *model*, *tool* and *loop*.

---

## Part 6 — Commit and push

```bash
git status      # should list observations.md and tools.py, and NOT .env
git add observations.md tools.py
git commit -m "Lab 0 complete"
git push
```

!!! warning

    If `.env` ever shows up in `git status`, stop. Don't commit; ask for help.

---

## Troubleshooting

??? failure "`mise: command not found`, or `uv`/`ollama` not found after `mise use`"

    Close and reopen the terminal. On Windows, sign out and back in. If it still fails, run `mise doctor` and look for warnings about activation or shims.

??? failure "`uv: command not found`"

    Close and reopen the terminal. On Windows, sign out and back in. On Linux, run `source $HOME/.local/bin/env`, or check that `~/.local/bin` is on your `PATH`.

??? failure "`FAIL Ollama is running`"

    Start the Ollama app. Or run `ollama serve` in a second terminal and leave it open; if you installed Ollama with mise, this is the only way to start it. On Linux, start the service with `sudo systemctl start ollama` and check it with `systemctl status ollama`.

??? failure "`FAIL Model … is downloaded`"

    Run `ollama pull qwen3:1.7b`. The name in `.env` must match `ollama list` exactly.

??? failure "`FAIL HF_TOKEN is set`, but you set it"

    The line must be exactly `HF_TOKEN=hf_...` with no spaces or quotes, in a file named `.env` (not `.env.txt`; Windows hides extensions) in the repo's top folder.

??? failure "`FAIL Hugging Face accepts your token`"

    The token was copied incompletely or revoked. Make a new one.

??? failure "Everything is very slow"

    Close other apps, especially browsers. The first call after a pause reloads the model (10–30 seconds). Check memory with `ollama ps`.

??? failure "More than 4 GB of memory in use"

    Lower `NUM_CTX` in `.env` to `4096`, or switch to `qwen3:0.6b`.

??? failure "The model answers without calling any tools"

    That's a small-model failure, not a setup bug. Rerun it, rephrase, or note it as an observation.

??? failure "You want a fresh database"

    ```bash
    uv run data/build_db.py
    ```
