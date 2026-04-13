# How to Use Claude the Smart Way

*A short guide for AI researchers.*

## The misread

Most people meet Claude Code and file it mentally as "a fancy autocomplete." That framing is why they get mediocre results. Claude Code isn't a faster keyboard — it's a collaborator you delegate to. The distinction matters most for researchers, because your bottleneck was never typing. It's attention: the scarce, finite budget you spend on which experiment to run, which bug is real, which paper is worth reading. Used well, Claude Code gives you that attention back. Used poorly, it just generates more things for you to review.

## The mindset shift

Stop thinking of yourself as "prompting a tool." Start thinking of yourself as onboarding a new teammate — one with no context, no memory of yesterday, and infinite patience. Every interaction is day one for them. This reframe has three immediate consequences:

- **Invest in context before you ask for output.** A teammate who understands the repo, the goal, and the constraints produces work you can use. A teammate who doesn't produces work you have to rewrite. Front-load the briefing.
- **Think in tasks, not keystrokes.** Don't ask Claude to "write a for loop." Ask it to "add a validation step that flags runs where eval loss diverges from train loss by more than 2x." The bigger and better-specified the task, the more leverage you get out of each turn.
- **Verify before you trust — especially numbers.** Claude is confident, not correct. For a researcher that's an existential caveat: your entire job is numbers, and a plausible-looking table is worse than no table at all. Check the ones that matter yourself.

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

## How to run things
- Train: `python train.py --config configs/base.yaml`
- Eval:  `python eval.py --ckpt checkpoints/latest.pt`
- Env:   `conda activate <envname>` (Python 3.11, CUDA 12.1, torch 2.3.1 pinned)

## Where things live
- `configs/`   — experiment configs; `base.yaml` is the canonical starting point
- `data/`      — raw in `data/raw/`, processed in `data/processed/v2` (v1 is stale)
- `src/`       — model and training code; read this, don't touch `legacy/`
- `scripts/`   — one-off analysis; not source of truth
- `notebooks/` — exploration only; never import from these

## Gotchas
- Eval silently skips NaN-loss batches; check `logs/skipped.txt` after every run
- Do NOT upgrade torch past 2.3.1 — breaks the custom CUDA kernel in `src/ops/`
- Seeds are set in `configs/base.yaml`, not in the training script
- W&B project is `myproject-dev`; `myproject` is reserved for paper runs
```

Fill in the blanks honestly. The gotchas section is where this file earns its keep — every line there is a mistake you don't have to make twice.

### Subagents — parallel work without context bloat

Spawn independent agents to work on tasks in parallel without polluting your main context. "Search the entire codebase for every place we set a learning rate" is a subagent job — it returns a short answer without flooding your main conversation with thousands of lines of grep output. Researchers should think of subagents the way they think of background processes: anything independent, dispatch it.

### Skills — reusable procedural knowledge

Skills are reusable procedural knowledge Claude invokes on demand. Instead of re-explaining every session how you want TDD, debugging, or brainstorming to work, a skill captures it once.

The [**superpowers**](https://github.com/obra/superpowers) plugin bundles battle-tested skills for exactly these workflows — brainstorming, writing plans, systematic debugging, TDD, verification-before-completion. Install it, read what's in it, and your baseline quality jumps noticeably without any extra effort on your part. Follow the install instructions in the repo; it takes under a minute, and it is the single highest-leverage change you can make after finishing this article.

### Cross-model code review

One model reviewing its own work has blind spots. Two different models rarely share the same blind spot. The [**codex-plugin-cc**](https://github.com/openai/codex-plugin-cc) plugin lets you hand a diff to OpenAI's Codex for a second opinion from inside Claude Code. Researchers already trust ensembles over individual models when making predictions — the same logic applies to code review. It's the cheapest way to catch the bug Claude confidently wrote and then confidently approved.

### More skills and plugins worth knowing about

Superpowers and codex-plugin-cc are the two most load-bearing installs, but the ecosystem is moving fast and the highest-leverage choice for *your* workflow may be something more specialized. A few entry points worth bookmarking:

**The directory.** Instead of hardcoding a list that will go stale, point yourself at the curated index:

- [**awesome-claude-code**](https://github.com/hesreallyhim/awesome-claude-code) — the canonical index of skills, hooks, slash-commands, plugins, and agent orchestrators. If you spend five minutes here every couple of weeks, you'll notice the ecosystem before it notices you.

**Paper and arXiv tools for researchers specifically.** A handful of MCP servers let Claude fetch and read papers directly inside your session, so your literature-triage workflow stops being "manually dump PDFs into a directory" and becomes "ask Claude to go fetch them."

- [**arxiv-mcp-server**](https://github.com/blazickjp/arxiv-mcp-server) — search and analyze arXiv papers from inside Claude Code. The most battle-tested option if you live on arXiv.
- [**paper-search-mcp**](https://github.com/openags/paper-search-mcp) — broader coverage: arXiv, PubMed, bioRxiv, medRxiv, Google Scholar. Pick this one if your field isn't only CS.
- [**arxiv-latex-mcp**](https://github.com/takashiishida/arxiv-latex-mcp) — fetches a paper's LaTeX source instead of the PDF, so Claude reads equations precisely instead of mangling them through OCR. Niche, but uniquely valuable for theory papers where the math is the whole point.

Install one paper tool and one directory bookmark and you are already ahead of most people using Claude Code today.

## Researcher workflows in practice

Here is what the above looks like applied to work you actually do. Each workflow comes with a prompt you can copy verbatim.

**Reproducing a baseline from a messy repo.** Point Claude at the repo and say:

> "Read this repo and draft a `CLAUDE.md` summarizing what it is, how to run training and eval, where the data lives, and the three biggest things a new collaborator would trip on. Then draft a spec for reproducing Table 2 of the paper: hardware, seeds, configs, and success = within 0.3 points of the reported numbers. Dispatch subagents to read `src/`, `configs/`, and `scripts/` in parallel."

You spend your attention on the decisions — not on tracing imports.

**Debugging a training run.** The worst move is asking Claude to "fix it." The right move is to spec the bug first:

> "Here's the symptom: <paste logs>. Here's what I've already ruled out: <list>. Before proposing any fix, write a short bug spec: observed vs. expected behavior, candidate hypotheses ranked by likelihood, and the cheapest experiment that would falsify each one. We'll test them one at a time."

When Claude proposes a fix, run the diff through the codex plugin for a second opinion before you touch a production training run.

**Literature triage.** You have forty papers and four hours. Dump the PDFs into a directory and:

> "Spawn a subagent per PDF in `papers/`. Each subagent summarizes one paper against this template: (1) problem, (2) method in two sentences, (3) headline result, (4) relevance to our work on <topic>, (5) one-line verdict — must-read / skim / skip. Collect the results into a single markdown table sorted by relevance."

You get back a filterable digest in the time it used to take you to skim five abstracts.

**Data and eval wrangling.** Custom metrics, HuggingFace datasets with quirky schemas, eval harnesses that disagree subtly with each other — this is tedious code where Claude shines. The catch is that it's also code whose bugs produce plausible-looking numbers. Always pair this kind of task with a sanity check:

> "Before running this metric on the real dataset, construct a tiny synthetic example with a known answer I can compute by hand. Run the metric on that example first and show me the output. Only then run it on the real data."

This is non-negotiable. Numbers you don't verify are numbers you'll regret citing.

## Start here: five things to do before your next session

If you take nothing else from this article, do these five before you open Claude Code next:

1. **Install [superpowers](https://github.com/obra/superpowers).** Battle-tested skills for brainstorming, planning, debugging, TDD, and verification. The single highest-leverage install on this page.
2. **Install [codex-plugin-cc](https://github.com/openai/codex-plugin-cc).** One-command setup, and the next time Claude writes a non-trivial diff, you get a second opinion from a different model for free.
3. **Write a `CLAUDE.md`** for your current project. Paste the template above, spend ten minutes filling it in, and commit it. The difference shows up on the very next session.
4. **Pick one upcoming task and spec it first.** Don't write code. Have Claude draft the spec and plan; read them, push back, approve. Notice how much of the task was actually unclear until you wrote it down.
5. **Dispatch one subagent this week.** Anything independent: a codebase search, a paper summary, a config audit. Feel what it's like to get a clean answer back without your main conversation bloating up.

Do those five and you've already changed how you use the tool.

## Closing

The real unlock from Claude Code isn't speed. It's attention. Every task you hand off cleanly is attention reclaimed — attention you can spend on the parts of research that genuinely need a human: taste, judgment, the next question to ask, the result that doesn't fit and probably means something.

Use the tool well, and the shift is noticeable: your days stop being about typing and start being about deciding. That's the smart way to use Claude.
