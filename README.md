# How to Use Claude the Smart Way

*A short guide for AI researchers.*

## The misread

Most people meet Claude Code and file it mentally as "a fancy autocomplete." That framing is why they get mediocre results. Claude Code isn't a faster keyboard — it's a collaborator you delegate to. The distinction matters most for researchers, because your bottleneck was never typing. It's attention: the scarce, finite budget you spend on which experiment to run, which bug is real, which paper is worth reading. Used well, Claude Code gives you that attention back. Used poorly, it just generates more things for you to review.

## The mindset shift

Stop thinking of yourself as "prompting a tool." Start thinking of yourself as onboarding a new teammate — one with no context about your project, no memory of yesterday's conversation, and infinite patience. Every interaction is day one for them. This reframe has three immediate consequences:

- **Invest in context before you ask for output.** A teammate who understands the repo, the goal, and the constraints produces work you can use. A teammate who doesn't produces work you have to rewrite. Front-load the briefing.
- **Think in tasks, not keystrokes.** Don't ask Claude to "write a for loop." Ask it to "add a validation step that flags runs where eval loss diverges from train loss by more than 2x." The bigger and better-specified the task, the more leverage you get out of each turn.
- **Verify before you trust — especially numbers.** Claude is confident, not correct. For a researcher that's an existential caveat: your entire job is numbers, and a plausible-looking table is worse than no table at all. Check the ones that matter yourself.

## Spec → Plan → Code

The researcher's instinct is "just try it." Paste the pseudocode from the paper, run it, see what happens. That works for toy changes and falls apart on anything real — three hours later you're deep in a refactor you never agreed to, with no clear idea what "done" looks like.

The discipline catching on is simple: write a **spec** before a plan, and a **plan** before code.

- A **spec** answers *what* and *why*: what are we building, what counts as success, what's explicitly out of scope.
- A **plan** answers *how*: what files change, in what order, what could break.
- The **code** is the mechanical execution of an approved plan.

Each stage is cheap to revise and expensive to skip. Claude Code is unusually good at all three — draft the spec with it, argue with it about the spec, then have it turn the approved spec into a concrete plan before a single line of code gets written. For research this maps cleanly onto how you already think: the spec is the experiment you're proposing, the plan is the runbook, the code is the implementation. As a bonus, a written spec is the artifact you'll actually want in your lab notebook six weeks from now, when you've forgotten why you made the choices you made.

## Power features worth the investment

A handful of features are worth learning on day one. Each one pays for itself the first time you use it.

**`CLAUDE.md`.** A plain-text file at the root of your repo that Claude reads automatically on every session. Put the things a new collaborator would need: what the project is, how to run it, which directories matter, which gotchas have already burned you. Ten minutes of writing here saves hours of re-explaining later, and it compounds every session.

**Subagents.** Spawn independent agents to work on tasks in parallel without polluting your main context. "Search the entire codebase for every place we set a learning rate" is a subagent job — it returns a short answer without flooding your main conversation with thousands of lines of grep output. Researchers should think of subagents the way they think of background processes: anything independent, dispatch it.

**Skills.** Reusable procedural knowledge Claude invokes on demand. Instead of re-explaining every session how you want TDD, debugging, or brainstorming to work, a skill captures it once. The [**superpowers**](https://github.com/obra/superpowers) plugin bundles battle-tested skills for exactly these workflows — brainstorming, writing plans, systematic debugging, TDD, verification-before-completion. Install it, read what's in it, and your baseline quality jumps noticeably without any extra effort on your part.

**Cross-model code review.** One model reviewing its own work has blind spots. Two different models rarely share the same blind spot. The [**codex-plugin-cc**](https://github.com/openai/codex-plugin-cc) plugin lets you hand a diff to OpenAI's Codex for a second opinion from inside Claude Code. Researchers already trust ensembles over individual models when making predictions — the same logic applies to code review. It's the cheapest way to catch the bug Claude confidently wrote and then confidently approved.

## Researcher workflows in practice

Here is what the above looks like applied to work you actually do.

**Reproducing a baseline from a messy repo.** Start by pointing Claude at the repo and asking it to write a `CLAUDE.md` summarizing what's there — entry points, configs, data assumptions. Then have it draft a spec for your reproduction: what you're running, on what hardware, with what seed, with success defined as matching Table 2 within 0.3 points. Dispatch subagents to read different subdirectories in parallel. Let Claude draft the runbook. You spend your attention on the decisions — not on tracing imports.

**Debugging a training run.** OOMs, NaN losses, silently broken logging — every researcher knows this pain. The worst move is asking Claude to "fix it." The right move is to spec the bug first: what's the observed behavior, what's the expected behavior, what you've already ruled out. Then let Claude form hypotheses and test them one at a time, systematically, instead of shotgunning fixes. When it proposes a change to your training code, run the diff through the codex plugin for a second opinion before you touch a production run.

**Literature triage.** You have forty papers and four hours. Dump the PDFs into a directory, spawn subagents to summarize them in parallel against a structured template (problem, method, result, why-it-matters-to-you). You get back a filterable digest in the time it used to take you to skim five abstracts. The papers you actually need to read carefully, you read carefully. The rest, you've at least indexed — and next time you wonder "didn't someone try this already?", you have an answer.

**Data and eval wrangling.** Custom metrics, HuggingFace datasets with quirky schemas, eval harnesses that disagree subtly with each other — this is tedious code where Claude shines. The catch is that it's also code whose bugs produce plausible-looking numbers. Always pair this kind of task with a sanity check: ask Claude to compute the metric on a tiny known-answer case before trusting it on your real data. This is non-negotiable. Numbers you don't verify are numbers you'll regret citing.

## Closing

The real unlock from Claude Code isn't speed. It's attention. Every task you hand off cleanly is attention reclaimed — attention you can spend on the parts of research that genuinely need a human: taste, judgment, the next question to ask, the result that doesn't fit and probably means something.

Use the tool well, and the shift is noticeable: your days stop being about typing and start being about deciding. That's the smart way to use Claude.
