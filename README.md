# How to Use Claude the Smart Way

Most people meet Claude Code and file it mentally as "a fancy autocomplete." That framing is why they get mediocre results. Claude Code isn't a faster keyboard — it's a collaborator you delegate to. The distinction matters most for researchers, because your bottleneck was never typing. It's attention: the scarce, finite budget you spend on which experiment to run, which bug is real, which paper is worth reading. Used well, Claude Code gives you that attention back. Used poorly, it just generates more things for you to review.

## Spec → Plan → Code

The researcher's instinct is "just try it." Paste the pseudocode from the paper, run it, see what happens. That works for toy changes and falls apart on anything real — three hours later you're deep in a refactor you never agreed to, with no clear idea what "done" looks like.

The discipline catching on is simple: write a **spec** before a plan, and a **plan** before code.

- A **spec** answers *what* and *why*: what are we building, what counts as success, what's explicitly out of scope.
- A **plan** answers *how*: what files change, in what order, what could break.
- The **code** is the mechanical execution of an approved plan.

Each stage is cheap to revise and expensive to skip. For research this maps cleanly onto how you already think: the spec is the experiment you're proposing, the plan is the runbook, the code is the implementation. As a bonus, a written spec is the artifact you'll actually want in your lab notebook six weeks from now, when you've forgotten why you made the choices you made.

**Try this on your next task.** Instead of opening Claude and typing "implement X," start with:

> "Before writing any code, draft a one-page spec for X: what we're building, success criteria, explicit non-goals, and the riskiest assumptions. Ask me questions until you're confident the spec is right."

Then, once the spec is approved:

> "Now turn this spec into a step-by-step implementation plan. For each step, list the files you'll touch and what could go wrong. Do not write code yet."

You'll feel the difference within one task.

## Power features worth the investment

A handful of features are worth learning on day one. Each one pays for itself the first time you use it.

### `CLAUDE.md` — your project's briefing doc

A plain-text file at the root of your repo that Claude reads automatically on every session. Put the things a new collaborator would need: what the project is, how to run it, which directories matter, which gotchas have already burned you. Ten minutes of writing here saves hours of re-explaining later, and it compounds every session.

Here's a starter template you can paste into your own repo and edit in five minutes:

```markdown
# Project: <name>

## What this is
One paragraph: the research question, the approach, and what counts as success.

## Local environment
- **GPUs:** 8× A800 40GB. Use whichever are idle — check with `nvidia-smi` before launching a run.
- **Conda env:** `clsea` is already set up; `conda activate clsea`. Do not create a new env.
- **Versions:** Python 3.11, CUDA 12.1, torch 2.3.1 (pinned — see Gotchas).

## How to run things
- Train: `python train.py --config configs/base.yaml`
- Eval:  `python eval.py --ckpt checkpoints/latest.pt`

## Where things live
- `configs/`   — experiment configs; `base.yaml` is the canonical starting point
- `data/`      — raw in `data/raw/`, processed in `data/processed/v2` (v1 is stale)
- `src/`       — model and training code; read this, don't touch `legacy/`
- `scripts/`   — one-off analysis; not source of truth
- `notebooks/` — exploration only; never import from these

## Network
We're behind the Great Firewall. If `pip`, `git`, HuggingFace, or any OpenAI/Google endpoint hangs, run `clashon` to enable the proxy and retry. A hang is almost always a network issue, not a bug — don't debug the wrong problem.

## LLM API access
API keys for Qwen / GPT / Gemini / Claude live in `.env` at the repo root. Load them with `python-dotenv` or `source .env`. Never hardcode a key in a script, and never commit `.env`.

## Git
Remote is `github.com/ivowang/how-to-use-claude-the-smart-way`. Commit and push regularly; prefer small, self-contained commits over a single end-of-day dump.

## Gotchas
- Eval silently skips NaN-loss batches; check `logs/skipped.txt` after every run
- Do NOT upgrade torch past 2.3.1 — breaks the custom CUDA kernel in `src/ops/`
- Seeds are set in `configs/base.yaml`, not in the training script
- W&B project is `myproject-dev`; `myproject` is reserved for paper runs
```

Fill in the blanks honestly. The *Local environment*, *Network*, and *Gotchas* sections are where this file earns its keep — each line is either a mistake you don't have to make twice or a "why isn't anything working?" you don't have to rediscover from scratch.

### Subagents — parallel work without context bloat

Spawn independent agents to work on tasks in parallel without polluting your main context. A subagent runs in its own conversation, absorbs the noise (file contents, grep output, failed attempts), and hands you back a clean final answer. Researchers should think of subagents the way they think of background processes: anything independent, dispatch it.

**When to reach for one.** The test is simple. If a task is (a) *independent* — you don't need to argue with Claude mid-task — and (b) *noisy* — doing it inline would flood the main conversation with hundreds of lines of file contents or logs — it belongs in a subagent. Pattern-match on these:

- "Find every place in the repo that sets a learning rate, and give me a table of filename, line, and value."
- "Read these twelve candidate files and tell me which one contains the metric definition we're looking for."
- "Run the test suite and report only which tests failed and the one-line reason for each."
- "Summarize what this 800-line YAML config actually controls, grouped by subsystem."
- "Audit all `*.ipynb` files in `notebooks/` and list any that import from `src/legacy/`."

**How to dispatch one.** You do not need special syntax — you ask for it. A prompt that works:

> "Dispatch a subagent to answer this: <question>. The subagent should <read these files / run this command / search for this pattern>. Return only the final answer plus file/line references. Do not include intermediate file contents or tool output in the reply."

The last sentence is the important part. Without it, the subagent may dutifully paste everything it read into its response, which defeats the whole purpose. Tell it explicitly to absorb the noise and hand you back only the signal.

**Dispatching several at once.** The real multiplier is parallelism. Two or four subagents working at the same time finish in roughly the time of one:

> "Dispatch four subagents in parallel. Subagent 1: audit `src/` for learning-rate settings. Subagent 2: audit `configs/` for the same. Subagent 3: check `scripts/` for any hardcoded overrides. Subagent 4: check `notebooks/` for ad-hoc experiments. Each returns a short table. Collect all four into one summary when they're done."

**When NOT to use one.** If you need to iterate with Claude on the answer — argue, push back, refine — keep it in the main conversation. Subagents are for one-shot lookups and bulk work, not dialogue. And don't dispatch a subagent for something so small you could just answer it inline in three lines; the framing overhead isn't worth it.

### Skills and plugins — reusable procedural knowledge

Skills and plugins are reusable capabilities Claude invokes on demand. Instead of re-explaining every session how you want TDD, debugging, or literature search to work, you install a skill once and it is available every session afterward. This is where the compounding value of the ecosystem lives — and it is worth spending real time here.

**The one install you should not skip.**

- [**superpowers**](https://github.com/obra/superpowers) — the most load-bearing install in this article. Bundles battle-tested skills for brainstorming, writing plans, systematic debugging, TDD, and verification-before-completion. Installs in under a minute, and your baseline quality jumps noticeably without any extra effort on your part.

**Official Anthropic plugins** ([anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official)). A curated set of first-party plugins. The ones most relevant to research work:

- [**feature-dev**](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/feature-dev) — a structured spec → plan → implement → verify workflow, packaged as a plugin. Pairs directly with the discipline described above.
- [**code-simplifier**](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/code-simplifier) — reviews changed code for reuse, quality, and efficiency, then fixes what it finds. Keeps prototype code from rotting into spaghetti before a collaborator sees it.
- [**claude-md-management**](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/claude-md-management) — creates, audits, and maintains your CLAUDE.md. A clean CLAUDE.md improves every subsequent session, and this plugin removes the excuse of "I'll update it later."
- [**skill-creator**](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/skill-creator) — the meta-skill for writing new skills. The fastest path to capturing a repeatable research workflow (your preferred figure style, your dataset-loading boilerplate, your eval runner) as a reusable skill of your own.
- [**ralph-loop**](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/ralph-loop) — runs Claude Code in a recursive autonomous loop until a specification is satisfied. Designed for unattended overnight iteration: config sweeps, flaky-test matrices, grinding through TODO backlogs while you sleep.

**Browser automation and visual verification.**

- [**playwright-skill**](https://github.com/lackeyjb/playwright-skill) — lets Claude drive a real browser via Playwright to run E2E tests and verify visual output. Researchers building web-based demos, interactive dashboards, or paper artifact sites can have Claude check the thing actually works end-to-end instead of just compiling.

**Literature and papers.**

- [**paper-search-mcp**](https://github.com/openags/paper-search-mcp) — an MCP server giving Claude direct access to arXiv, PubMed, bioRxiv, medRxiv, and Google Scholar. Install this and your literature-triage workflow stops being "manually dump PDFs into a directory" and becomes "ask Claude to go fetch them."

**The directory.** The ecosystem moves fast, and any list in a README will go stale. Instead, bookmark the canonical index and browse it every couple of weeks:

- [**awesome-claude-code**](https://github.com/hesreallyhim/awesome-claude-code) — the community index of skills, hooks, slash-commands, plugins, and agent orchestrators. Spend five minutes there periodically and you will notice the ecosystem before it notices you.

**Quick-install script.** Paste this into your terminal to install the plugins above in one go. Requires the Claude Code CLI (`claude --version` should work). Safe to re-run; restart Claude Code afterward for the new plugins to activate.

```bash
#!/usr/bin/env bash
set -e

# 1. Add the plugin marketplaces we'll install from.
claude plugin marketplace add anthropics/claude-plugins-official
claude plugin marketplace add obra/superpowers-marketplace

# 2. Install superpowers — the highest-leverage single install.
claude plugin install superpowers@claude-plugins-official

# 3. Install the official Anthropic plugins recommended above.
for plugin in feature-dev code-simplifier claude-md-management skill-creator ralph-loop; do
  claude plugin install "${plugin}@claude-plugins-official"
done

# 4. Items that require a per-repo install step — follow their READMEs:
#    - codex-plugin-cc:  https://github.com/openai/codex-plugin-cc
#    - playwright-skill: https://github.com/lackeyjb/playwright-skill
#    - paper-search-mcp: https://github.com/openags/paper-search-mcp
#      (typically: `claude mcp add paper-search -- <command from repo README>`)

echo "Done. Restart Claude Code to activate the new plugins."
```

The last three items are left as manual steps because their install commands depend on details (package names, MCP transport, credentials) that change more often than is safe to hardcode here — the repos themselves have the current instructions and they take a minute each.

### Cross-model code review

One model reviewing its own work has blind spots. Two different models rarely share the same blind spot. The [**codex-plugin-cc**](https://github.com/openai/codex-plugin-cc) plugin lets you hand a diff to OpenAI's Codex for a second opinion from inside Claude Code. Researchers already trust ensembles over individual models when making predictions — the same logic applies to code review. It's the cheapest way to catch the bug Claude confidently wrote and then confidently approved.

**When to reach for it.** Every non-trivial diff that touches training code, eval code, or data pipelines. Not every one-line typo fix, but any change where a subtle bug would produce plausible-but-wrong numbers. Research code has a painful asymmetry: the cost of shipping a silent metric bug is orders of magnitude larger than the cost of a two-minute cross-review. Use it especially before you kick off a long training run or publish numbers.

**How to use it.** After Claude proposes a diff but before you apply it:

> "Send this diff to Codex for review. Ask it to look specifically for: off-by-one errors, silent type coercions, reduction-axis mistakes in tensor ops, and anything that would change the numeric output without raising an error. Come back with a ranked list of concerns and the specific lines each concern points at."

The "specific lines" clause matters — a vague "looks fine" is useless, and a specific "line 47 uses `dim=0` where I think you want `dim=-1`" is actionable.

**What to do with the review.** Don't treat Codex's verdict as ground truth either — it's just another model, with its own blind spots. The real signal is *disagreement*: when the two models flag different things, or one defends a choice the other questions, that is exactly the spot where you should read the code yourself and decide. The point is not consensus; the point is getting two different sets of eyes on the parts that most need them.

## How to write smart prompts

Everything above — the mindset shift, spec-first discipline, subagents, verification — comes together in the prompts you actually type. A good prompt is not magic phrasing. It is the compressed form of everything you already decided before touching the keyboard. The fastest way to get better at prompting is to look at two prompts for the same task side by side and ask: why does the second one work?

For each task below, both prompts take about the same amount of effort to type. The difference is not length. The difference is what the author already decided.

### Task: reproducing a baseline from a messy repo

**Bad prompt:**

> "Help me reproduce the results of this paper."

**Why it's bad.** "Reproduce" is undefined — which table, which numbers, within what tolerance? No context about the repo. No spec, no plan, just a slot-machine pull. Claude will either guess the wrong target or ask ten questions you should have answered upfront.

**Good prompt:**

> "Read this repo and draft a `CLAUDE.md` summarizing entry points, run commands, and data assumptions. Then draft a spec for reproducing Table 2 of the paper: hardware, seeds, configs, and success = within 0.3 points of the reported numbers. Push back if anything is ambiguous. Dispatch subagents to audit `src/`, `configs/`, and `scripts/` in parallel."

**Why it's good.** It names concrete artifacts (a CLAUDE.md, then a spec), defines success numerically, invites pushback so Claude doesn't silently guess, and parallelizes the noisy exploration across subagents. Every earlier principle in this article is compressed into this one prompt.

### Task: debugging a NaN loss

**Bad prompt:**

> "My loss is NaN, fix it."

**Why it's bad.** No context, no evidence, no ruling-out. It asks for a fix before a diagnosis, which is how you end up with three "fixes" that each paper over a different unverified guess. It also wastes Claude's turn on questions you already know the answers to.

**Good prompt:**

> "Symptom: training loss goes to NaN starting at step 3140 (log tail below). Already ruled out: FP16 overflow (we're in FP32), corrupted input batch (inspected step 3139's batch manually). Before proposing any fix, write a short bug spec — observed vs. expected behavior, candidate hypotheses ranked by likelihood, and the cheapest experiment that would falsify each. We'll test them one at a time."

**Why it's good.** It hands Claude the evidence, closes doors you've already checked so it doesn't repeat your work, and demands a spec before a fix — which forces explicit hypothesis ranking instead of guess-and-patch. This is the Spec → Plan → Code discipline from earlier, applied to a bug.

### Task: triaging forty papers

**Bad prompt:**

> "Read these forty papers and tell me which ones are important."

**Why it's bad.** "Important" is undefined — important to what? Processing forty PDFs sequentially will flood your main context with raw paper text and leave you with an unstructured blob you can't re-read in six weeks. No parallelism, no structure, no relevance criterion.

**Good prompt:**

> "Spawn one subagent per PDF in `papers/`. Each subagent summarizes its paper against this template: (1) problem, (2) method in two sentences, (3) headline result, (4) relevance to our work on <topic>, (5) verdict — must-read / skim / skip. Collect all results into a single markdown table sorted by relevance. Do not include raw paper text in the replies."

**Why it's good.** Parallelized via subagents, structured output you can filter later, an explicit relevance criterion that makes "important" concrete, and a noise-suppression clause that keeps your main context usable.

### Task: computing a custom metric on a real dataset

**Bad prompt:**

> "Compute <custom metric> on the eval set and report the number."

**Why it's bad.** You'll get a number. It will look plausible. You will cite it. And eventually someone will find the off-by-one in the metric implementation, and "plausible" will have cost you a revision cycle — or a corrigendum. Numbers you don't verify are numbers you'll regret.

**Good prompt:**

> "Before running <metric> on the real dataset, construct a tiny synthetic example with a known answer I can compute by hand. Run the metric on the synthetic example and show me the output. Only after I confirm it matches the hand-computed answer, run it on the real data."

**Why it's good.** It makes the mindset shift's third implication — verify before trusting, especially numbers — literal. A synthetic known-answer test is cheap; the regret of citing a wrong metric is not.

### Meta-prompting: have Claude help you write the prompt

Sometimes the hardest part of a task is figuring out how to ask. **Meta-prompting** is exactly what it sounds like: using Claude to improve the prompt you're about to send Claude. It sounds silly until you try it and notice your first draft was missing half the context the task actually needed.

Two forms that reliably work:

**1. Critique-and-rewrite.** You have a draft prompt — have Claude tear it apart before you run it.

> "Here's the prompt I'm about to run: <paste>. Don't execute it yet. Critique it: what is ambiguous, what context is missing that would change the answer, and what will you most likely hallucinate or get wrong if I run it as-is? Then rewrite it to fix those issues."

**2. Reverse elicitation.** You have a fuzzy goal — have Claude ask *you* the questions it would need answered to do the task well, before it does anything.

> "I want to investigate why our model underperforms on long-context inputs. Before doing anything, list the five questions you would need answered to run this investigation well — what evidence, what files, what metrics we should agree on upfront. I'll answer them, and then you'll write the prompt for the actual investigation."

**A practical case.** Say you want to run an ablation but aren't sure what's worth ablating. The naive prompt — "what should I ablate?" — gets a generic checklist anyone could have written. The meta-prompt — "before suggesting ablations, ask me the five questions about the model, data, and hypothesis that would most change your recommendation, then use my answers to propose a prioritized list" — gets a list tailored to your actual situation, because Claude now has the context it needed to give a useful answer. The five-minute detour of answering those questions upfront saves the hour of arguing with a generic list.

Meta-prompting is really just the mindset shift taken one step further: if you are onboarding a new teammate, sometimes the most productive thing you can do is let them interview you first.

## Closing

The real unlock from Claude Code isn't speed. It's attention. Every task you hand off cleanly is attention reclaimed — attention you can spend on the parts of research that genuinely need a human: taste, judgment, the next question to ask, the result that doesn't fit and probably means something.

Use the tool well, and the shift is noticeable: your days stop being about typing and start being about deciding. That's the smart way to use Claude.
