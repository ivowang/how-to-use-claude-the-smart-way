# How to Use Claude the Smart Way

[中文版 (Chinese)](README_ZH.md)

Most people try Claude Code, decide it's a fancy autocomplete, and go back to what they were doing. That's why they end up with mediocre results. The better way to think about Claude Code is as a collaborator you hand work to, not as a faster keyboard. This matters more for researchers than for almost anyone else, because typing was never what was slowing you down. Your real bottleneck is attention: figuring out which experiment to run next, which bug is actually a bug, which paper is worth an afternoon. Used well, Claude Code gives a lot of that attention back. Used badly, it just hands you more things to review.

## How to setup Claude Code before a work

A few things are worth doing up front.

### Write a CLAUDE.md

`CLAUDE.md` is a plain text file you drop at the root of your repo. Claude reads it automatically at the start of every session. Treat it like the onboarding doc you'd write for a new collaborator: what the project is, how to run things, which directories matter, which things have already bitten you. Ten minutes spent writing it saves a lot of re-explaining later, and every session afterward gets to start from that higher baseline.

If you're a researcher or a PhD student, your `CLAUDE.md` really should describe your local setup: GPUs, network config, Python env, and anything else Claude needs to know to actually run experiments on your machine. Here's a template you can lift and edit:

```markdown
# Project: <name>

## What this is
One paragraph: the research question, the approach, and what counts as success.

## Local environment
- **GPUs:** 8× A800 40GB. Use whichever are idle, check with `nvidia-smi` before launching a run.
- **Conda env:** `test` is already set up; `conda activate test`. Do not create a new env.
- **Versions:** Python 3.11, CUDA 12.1, torch 2.3.1 (pinned, see Gotchas).

## Network
We're behind the Great Firewall. If `pip`, `git`, HuggingFace, or any OpenAI/Google endpoint hangs, run `clashon` to enable the proxy and retry. A hang is almost always a network issue, not a bug, don't debug the wrong problem.

## How to run things
- Train: `python train.py --config configs/base.yaml`
- Eval:  `python eval.py --ckpt checkpoints/latest.pt`

## Where things live
- `configs/`  , experiment configs; `base.yaml` is the canonical starting point
- `data/`     , raw in `data/raw/`, processed in `data/processed/v2` (v1 is stale)
- `src/`      , model and training code; read this, don't touch `legacy/`
- `scripts/`  , one-off analysis; not source of truth
- `notebooks/`, exploration only; never import from these

## LLM API access
API keys for Qwen / GPT / Gemini / Claude live in `.env` at the repo root. Load them with `python-dotenv` or `source .env`. Never hardcode a key in a script, and never commit `.env`.

## Git
Remote is `github.com/ivowang/how-to-use-claude-the-smart-way`. Commit and push regularly; prefer small, self-contained commits over a single end-of-day dump.

## Gotchas
- Eval silently skips NaN-loss batches; check `logs/skipped.txt` after every run
- Do NOT upgrade torch past 2.3.1, breaks the custom CUDA kernel in `src/ops/`
- Seeds are set in `configs/base.yaml`, not in the training script
- W&B project is `myproject-dev`; `myproject` is reserved for paper runs
```

### Extend Claude with skills and plugins

Skills and plugins are reusable bits of capability that Claude can call on when it needs them. Install one once, and every future session has access to it. No more re-explaining how you want TDD to work, or debugging, or literature search. You just install the thing. This is probably where the ecosystem gives you the most back per minute of effort, and it's worth spending real time here.

**The one you shouldn't skip.**

- [**superpowers**](https://github.com/obra/superpowers) packages up a set of well-tested skills for brainstorming, writing plans, systematic debugging, TDD, and verification. Install takes under a minute. You'll notice the baseline quality jump afterward, and you don't have to do anything extra for it.

**Official Anthropic plugins** (from [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official)). A curated set of first-party plugins. The ones most worth installing for research work:

- [**feature-dev**](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/feature-dev), a structured spec → plan → implement → verify workflow, packaged as a plugin. The plugin version of the Spec → Plan → Code discipline we get into later.
- [**code-simplifier**](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/code-simplifier), reviews the code you've changed for readability, reuse, and efficiency, and then fixes what it finds. Good for keeping prototype code from going feral before a collaborator sees it.
- [**claude-md-management**](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/claude-md-management), creates, audits, and maintains your `CLAUDE.md`. A clean `CLAUDE.md` makes every session after it better, and this plugin takes away the usual excuse for letting it rot.
- [**skill-creator**](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/skill-creator), the meta-skill for writing new skills. The fastest way to turn the things you do on every project into something Claude just does for you.
- [**ralph-loop**](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/ralph-loop), runs Claude Code in a recursive loop until a spec is satisfied. Made for unattended overnight work: config sweeps, fighting a flaky test matrix, eating through a TODO list while you sleep.

**Browser automation and visual verification.**

- [**playwright-skill**](https://github.com/lackeyjb/playwright-skill), lets Claude drive an actual browser through Playwright. If you're shipping a web demo, an interactive dashboard, or a paper artifact site, this is how you get Claude to check that the thing *works* rather than just that the code compiled.

**Literature and papers.**

- [**paper-search-mcp**](https://github.com/openags/paper-search-mcp), an MCP server that gives Claude direct access to arXiv, PubMed, bioRxiv, medRxiv, and Google Scholar. Install it and you stop having to dump folders of PDFs at Claude. You tell it what you're looking for and it goes and finds them.

**A directory worth bookmarking.** The ecosystem moves fast, and any hardcoded list in a README will be out of date within a month. Better to bookmark the community index and come back to it every few weeks:

- [**awesome-claude-code**](https://github.com/hesreallyhim/awesome-claude-code), a community-maintained index of skills, hooks, slash commands, plugins, and agent orchestrators. Five minutes there every so often is enough to stay caught up.

**A quick install script.** Paste this into your terminal to install the plugins above in one shot. You'll need the Claude Code CLI (`claude --version` should work). Safe to re-run. Restart Claude Code afterward so the new plugins actually load.

```bash
#!/usr/bin/env bash
set -e

# 1. Add the plugin marketplaces we'll install from.
claude plugin marketplace add anthropics/claude-plugins-official
claude plugin marketplace add obra/superpowers-marketplace

# 2. Install superpowers, the highest-leverage single install.
claude plugin install superpowers@claude-plugins-official

# 3. Install the official Anthropic plugins recommended above.
for plugin in feature-dev code-simplifier claude-md-management skill-creator ralph-loop; do
  claude plugin install "${plugin}@claude-plugins-official"
done

# 4. Items that require a per-repo install step, follow their READMEs:
#    - codex-plugin-cc:  https://github.com/openai/codex-plugin-cc
#    - playwright-skill: https://github.com/lackeyjb/playwright-skill
#    - paper-search-mcp: https://github.com/openags/paper-search-mcp
#      (typically: `claude mcp add paper-search -- <command from repo README>`)

echo "Done. Restart Claude Code to activate the new plugins."
```

The last three aren't scripted because their install commands depend on details (package names, MCP transport, credentials) that change too often to safely hardcode here.

### Cross-check Claude with another model

One model reviewing its own work has blind spots. Two different models rarely share the *same* blind spots. The [**codex-plugin-cc**](https://github.com/openai/codex-plugin-cc) plugin lets you hand a diff to OpenAI's Codex for a second opinion without leaving Claude Code. Researchers already trust ensembles over single models when making predictions. Apply the same instinct to code review. It's about the cheapest way to catch the kind of bug Claude confidently writes and then confidently signs off on.

**When to use it.** Any non-trivial diff that touches training code, eval code, or a data pipeline. Not every one-line typo fix, but anything where a subtle bug would quietly produce wrong numbers. The asymmetry in research code is unforgiving: the cost of shipping a silent metric bug is much higher than the cost of a two-minute review. Use it especially before you kick off a long training run, and before you publish any numbers.

**How to use it.** After Claude writes a diff, and before you apply it:

> "Send this diff to Codex for review. Ask it to look specifically for: off-by-one errors, silent type coercions, reduction-axis mistakes in tensor ops, and anything that would change the numeric output without raising an error. Come back with a ranked list of concerns and the specific lines each concern points at."

The specific-lines clause is doing most of the work here. A vague verdict that the code looks fine is worth nothing. Something concrete, like a note that line 47 uses `dim=0` where you probably want `dim=-1`, is what you can actually act on.

**What to do with the review.** Don't treat Codex's verdict as ground truth either. It's just another model, with its own blind spots. What you actually want out of this is *disagreement*. When the two models flag different things, or one defends a choice the other questions, that's exactly where you should stop and read the code yourself. The goal isn't to get them to agree. It's to point two different pairs of eyes at the places that most need a second look.

### Have Claude keep a development journal

For any project that's going to span more than a handful of sessions, have Claude keep its own notes in a `docs/dev/` folder, one short file per thing worth remembering. Each file captures the symptom, what you tried, what finally worked, and the one-line takeaway. Here's a prompt that sets the habit on day one:

> "From now on, whenever we hit a non-trivial bug, debug a surprising behavior, or make a design decision with real tradeoffs, write a short note in `docs/dev/YYYY-MM-DD-<slug>.md` with: problem, root cause, what we tried, what ended up working, and a one-line lesson. Keep each file under a page. Before starting any new task in this repo, re-read the most recent notes in `docs/dev/` and tell me if anything there is relevant to what I'm about to ask."

Each note is a minute to write and basically free to maintain. But they stack up. A few dozen sessions in, you'll notice Claude starting to check its own old notes before repeating a debugging session you already ran, spotting patterns across bugs, and waving you off from decisions a past note warned against. In a modest but genuine sense, this is an agent self-evolving at the memory layer: every session ends with it slightly more experienced in *your* specific project than when that session started. You don't need fine-tuning or a long-term memory architecture for any of this. A folder of markdown files plus a prompt telling Claude to read it first gets you most of the way there.

## How to write smart prompts in a work

With the setup out of the way, you can actually start using Claude for tasks. The main way you'll interact with it is by writing prompts, and there's nothing magical about good ones. A good prompt is just the compressed form of everything you'd already figured out before you started typing.

### Spec -> Plan -> Code

The researcher's instinct is to just try it. Paste the pseudocode from the paper, run it, see what happens. That's fine for toy changes. It falls apart on anything real. Three hours in, you're deep in a refactor you never signed up for, with no clear idea what being done is even supposed to mean.

The discipline that's been catching on is simple: write a *spec* before the plan, and a *plan* before any code.

- A **spec** answers *what* and *why*. What are we building? What counts as done? What's explicitly out of scope?
- A **plan** answers *how*. Which files change, in what order, and what could break along the way?
- The **code** is just the mechanical execution of the plan, once you've approved it.

Each stage is cheap to rewrite and expensive to skip. For a researcher, the mapping is clean: the spec is the experiment you're proposing, the plan is the runbook, the code is the implementation. As a side benefit, six weeks from now, when you've forgotten why you made the choices you made, the spec is the thing you'll actually want open in your lab notebook.

**Try this on your next task.** Instead of opening Claude and typing *implement X*, start with:

> "Before writing any code, draft a one-page spec for X: what we're building, success criteria, explicit non-goals, and the riskiest assumptions. Ask me questions until you're confident the spec is right."

Once the spec is signed off:

> "Now turn this spec into a step-by-step implementation plan. For each step, list the files you'll touch and what could go wrong. Do not write code yet."

One task in, you'll feel the difference.

### Use subagents

Subagents let you spin off independent tasks in parallel without dragging noise into your main conversation. Each subagent runs in its own isolated chat, absorbs all the messy stuff (file contents, grep output, dead ends), and hands you back a clean final answer. Think of them the way you think of background processes: anything you can cut loose, cut it loose.

**When to reach for one.** The test is simple. A task belongs in a subagent if it is (a) *independent*, meaning you won't need to argue with Claude mid-task, and (b) *noisy*, meaning doing it inline would flood your main conversation with hundreds of lines of file contents or logs. A few things to pattern-match on:

- Find every place in the repo that sets a learning rate, and report a table of filename, line, and value.
- Read these twelve candidate files and tell me which one contains the metric definition we're looking for.
- Run the test suite and report only which tests failed, with the one-line reason for each.
- Summarize what this 800-line YAML config actually controls, grouped by subsystem.
- Audit all `*.ipynb` files in `notebooks/` and list any that import from `src/legacy/`.

**How to dispatch one.** No special syntax. You just ask for it. Something like:

> "Dispatch a subagent to answer this: <question>. The subagent should <read these files / run this command / search for this pattern>. Return only the final answer plus file/line references. Do not include intermediate file contents or tool output in the reply."

That last sentence is the one that matters. Leave it out and the subagent will dutifully dump everything it read back into its reply, which defeats the whole point. Tell it plainly: absorb the noise, hand me the signal.

**Dispatching several at once.** The real win is parallelism. Two or four subagents running at the same time finish in roughly the time of one:

> "Dispatch four subagents in parallel. Subagent 1: audit `src/` for learning-rate settings. Subagent 2: audit `configs/` for the same. Subagent 3: check `scripts/` for any hardcoded overrides. Subagent 4: check `notebooks/` for ad-hoc experiments. Each returns a short table. Collect all four into one summary when they're done."

**When not to use one.** If you need to go back and forth with Claude on the answer, whether that means arguing with it, pushing back, or iterating, keep the task in your main conversation. Subagents are for one-shot lookups and bulk work, not for dialogue. And don't dispatch one for something small enough that you could just answer it inline in three lines. The overhead of setting up a subagent isn't worth it for tiny tasks.

### Meta-Prompting

Sometimes the hardest part of a task is figuring out how to ask for it. Meta-prompting is exactly what it sounds like: you use Claude to improve the prompt you're about to send to Claude. It sounds silly. Try it once, and you'll notice your first draft was missing half the context the task actually needed.

Two patterns that reliably work.

**1. Critique and rewrite.** You've got a draft prompt. Have Claude tear it apart before you run it.

> "Here's the prompt I'm about to run: <paste>. Don't execute it yet. Critique it: what's ambiguous, what context is missing that would change the answer, and what will you most likely hallucinate or get wrong if I run it as-is? Then rewrite it to fix those issues."

**2. Reverse elicitation.** You've got a fuzzy goal. Have Claude interview *you* for what it needs before it does anything.

> "I want to investigate why our model underperforms on long-context inputs. Before doing anything, list the five questions you'd need answered to run this investigation well: what evidence, what files, what metrics we should agree on up front. I'll answer them, then you'll write the prompt for the actual investigation."

**A real case.** Say you want to run an ablation but you're not sure what's worth ablating. Asking Claude what to ablate, cold, gets you a generic checklist anyone could have written. Asking Claude to first interview you about the model, the data, and the hypothesis, and then to propose a prioritized list based on your answers, gets you something tailored to your actual situation. By the time Claude answers, it has the context it needed to be useful. The five minutes you spend answering those questions up front saves you an hour of arguing with a generic list.

Meta-prompting takes the teammate framing one step further. If you are onboarding a new teammate, sometimes the most useful thing you can do is let them interview you first.

### Prompt examples

The fastest way to get better at prompting is to put two prompts for the same task side by side and ask: why does the second one work? It isn't that the good one is longer. It's that the person who wrote it had already decided more before they started typing.

#### Example.1 Ask Claude to reproduce a baseline

**Bad prompt:**

> "Help me reproduce the results of this paper."

**Why it's bad.** The word *reproduce* isn't defined. Which table? Which numbers? Within what tolerance? There's no context about the repo, no spec, no plan, just a pull on a slot machine. Claude will either guess the wrong target or fire back ten questions you should have answered yourself before asking.

**Good prompt:**

> "Read this repo and draft a `CLAUDE.md` summarizing entry points, run commands, and data assumptions. Then draft a spec for reproducing Table 2 of the paper: hardware, seeds, configs, and success = within 0.3 points of the reported numbers. Push back if anything is ambiguous. Dispatch subagents to audit `src/`, `configs/`, and `scripts/` in parallel."

**Why it's good.** It names concrete deliverables: first a `CLAUDE.md`, then a spec. It pins success to a number. It tells Claude to push back instead of silently guessing. And it parallelizes the noisy exploration with subagents. Basically everything else in this article, compressed into a single prompt.

#### Example.2 Ask Claude to debug a NaN loss

**Bad prompt:**

> "My loss is NaN, fix it."

**Why it's bad.** No context, no evidence, no mention of what you've already ruled out. It asks for a fix before a diagnosis, which is how you end up with three different attempted fixes that each paper over a different unverified guess. It also wastes Claude's turn asking you questions you already know the answers to.

**Good prompt:**

> "Symptom: training loss goes to NaN starting at step 3140 (log tail below). Already ruled out: FP16 overflow (we're in FP32), corrupted input batch (inspected step 3139's batch manually). Before proposing any fix, write a short bug spec, observed vs. expected behavior, candidate hypotheses ranked by likelihood, and the cheapest experiment that would falsify each. We'll test them one at a time."

**Why it's good.** It hands Claude the evidence, closes the doors you've already checked so Claude doesn't redo your work, and forces a spec before a fix, which means explicit hypothesis ranking instead of guess-and-patch. It's the Spec → Plan → Code discipline from earlier in this section, applied to a bug.

#### Example.3 Ask Claude to triage forty papers

**Bad prompt:**

> "Read these forty papers and tell me which ones are important."

**Why it's bad.** The word *important* isn't defined. Important for what? Forty PDFs read in sequence will flood your main context with raw paper text and leave you with a giant unstructured blob you can't come back to in six weeks. No parallelism, no structure, no relevance criterion.

**Good prompt:**

> "Spawn one subagent per PDF in `papers/`. Each subagent summarizes its paper against this template: (1) problem, (2) method in two sentences, (3) headline result, (4) relevance to our work on <topic>, (5) verdict, must-read / skim / skip. Collect all results into a single markdown table sorted by relevance. Do not include raw paper text in the replies."

**Why it's good.** It parallelizes the work across subagents. The output is structured, so you can actually filter it later. The relevance field turns the fuzzy notion of importance into something concrete. And the noise-suppression clause keeps your main context usable when the subagents report back.

#### Example.4 Ask Claude to compute a custom metric

**Bad prompt:**

> "Compute <custom metric> on the eval set and report the number."

**Why it's bad.** You'll get a number. It'll look plausible. You'll cite it. And eventually someone will find an off-by-one in the metric implementation, and that plausible-looking number will have cost you a revision round. Or worse, a corrigendum. Numbers you don't verify are numbers you'll regret citing.

**Good prompt:**

> "Before running <metric> on the real dataset, construct a tiny synthetic example with a known answer I can compute by hand. Run the metric on the synthetic example and show me the output. Only after I confirm it matches the hand-computed answer, run it on the real data."

**Why it's good.** It takes the rule of verifying before you trust, especially for numbers, and makes it completely literal. A synthetic, hand-checkable test case is cheap. Citing a wrong metric is not.

## Close

The real win from Claude Code isn't speed. It's attention. Every task you hand off cleanly is attention given back to you. You can spend it on the parts of research that actually need a human: taste, judgment, the next question to ask, the result that doesn't fit the model and probably means something.

Use the tool well and the shift becomes obvious. Your days stop being about typing and start being about deciding. That's what it means to use Claude the smart way.

## Citation

If this article has been helpful to your research work, feel free to cite it:

```bibtex
@misc{wang2026claudesmart,
  author       = {Ziyi Wang},
  title        = {How to Use Claude the Smart Way},
  year         = {2026},
  howpublished = {\url{https://github.com/ivowang/how-to-use-claude-the-smart-way}}
}
```
