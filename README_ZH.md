# 使用 Claude 的智慧

[English](README.md)

大多数人第一次用 Claude Code，都会把它归类为一个高级点的自动补全，然后就没然后了。这也是为什么他们最后拿到的结果都一般般。正确的理解应该是：Claude Code 不是一个更快的键盘，它是一个您可以把活儿交出去的协作者。这件事对做研究的人尤其重要，因为拖慢您的从来就不是打字，而是注意力：该跑哪个实验、哪个 bug 才是真的 bug、哪篇论文值得花一个下午去读。用对了，Claude Code 能把您的注意力还给您；用错了，它只是给您造出更多需要您审的东西。

## 在开始工作前如何配置好 Claude Code

有几件事值得您在一开始就做好。每一件都会在您第一次真正需要用到它的时候把成本赚回来。

### 写一份 CLAUDE.md

`CLAUDE.md` 是一份放在 repo 根目录下的纯文本文件，Claude 每次会话开始时都会自动读一遍。您可以把它当成您写给一个新协作者的上手文档：这个项目是什么、怎么跑、哪些目录是重要的、哪些坑别人已经帮您踩过了。花十分钟写这份文件，后面能省下很多次重新解释的时间，而且之后的每一次会话都能从这份文档的基础上起步，而不是从零开始。

如果您是做 AI 研究的，不管是研究员还是博士生，您的 `CLAUDE.md` 里还应该特别写清楚本地环境：GPU 情况、网络代理、Python 环境，以及任何 Claude 在您这台机器上实际跑实验时需要知道的东西。下面这份模板您可以直接拿去改：

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

### 用 skills 和 plugins 扩展 Claude

Skills 和 plugins 是 Claude 可以按需调用的可复用能力。安装一次之后，每次会话都能用上它。您不用每次都重新跟它讲您想要的 TDD、调试或文献检索流程是什么样，直接把能力装进去就行。整个生态里单位时间回报率最高的地方就是这里，值得您多花点时间逛一逛。

**一个不能跳过的安装。**

- [**superpowers**](https://github.com/obra/superpowers) 打包了一组经过充分打磨的 skills：brainstorming、writing plans、systematic debugging、TDD、verification-before-completion。一分钟以内装完，装完之后您会明显感觉基线质量上了一个台阶，而您并不需要为此多做任何事。

**官方 Anthropic plugins**（在 [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official) 上）。一套精挑细选的一方 plugin。和研究工作最相关的几个：

- [**feature-dev**](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/feature-dev)，一个结构化的 spec → plan → implement → verify 工作流，做成了 plugin。说白了就是本文后面会讲到的 Spec → Plan → Code 纪律的 plugin 版。
- [**code-simplifier**](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/code-simplifier)，审查您改过的代码，看可复用性、质量、效率，然后把发现的问题直接修掉。能防止原型代码在合作者看到之前就烂成一团乱麻。
- [**claude-md-management**](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/claude-md-management)，帮您创建、审计、维护 `CLAUDE.md`。一份干净的 `CLAUDE.md` 会让之后每一次会话的质量都更好，而这个 plugin 直接帮您省掉了"回头再整"的那种拖延。
- [**skill-creator**](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/skill-creator)，写新 skill 用的元 skill。如果您有一个每个项目都要重复做的事情，比如您偏好的画图风格、您的数据加载样板、您的 eval runner，想把它沉淀成一个可复用的 skill，它是最快的那条路。
- [**ralph-loop**](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/ralph-loop)，让 Claude Code 递归地、自主地循环跑下去，直到某个 spec 被满足为止。专门为那种您不想一直盯着的通宵任务设计：配置扫参、啃一个不稳定的测试矩阵、睡觉的时候把 TODO 列表磨完。

**浏览器自动化和视觉验证。**

- [**playwright-skill**](https://github.com/lackeyjb/playwright-skill)，让 Claude 通过 Playwright 去驱动一个真实的浏览器。如果您在搭 web demo、交互 dashboard 或者 paper artifact 站点，靠它您可以让 Claude 端到端地确认这东西真的能用，而不只是代码能编译过去。

**文献和论文。**

- [**paper-search-mcp**](https://github.com/openags/paper-search-mcp)，一个 MCP server，让 Claude 能直接访问 arXiv、PubMed、bioRxiv、medRxiv 和 Google Scholar。装了它之后，您不再需要手动把 PDF 丢到一个文件夹里去喂给 Claude，您只要告诉它您在找什么，它自己就会去取。

**一个值得收藏的索引。** 这个生态变化很快，任何写死在 README 里的列表，一个月后差不多就过期了。与其自己维护一份清单，不如把社区的索引收进书签，每隔几周回去逛一次：

- [**awesome-claude-code**](https://github.com/hesreallyhim/awesome-claude-code)，社区维护的 skills、hooks、slash-commands、plugins 和 agent orchestrators 索引。隔一段时间花五分钟翻一下，就足够不落伍了。

**快速安装脚本。** 下面这段脚本粘贴到终端里就能一次性把上面推荐的 plugin 都装上。您需要已经装好了 Claude Code CLI（`claude --version` 能跑通就行）。脚本可以重复运行；装完之后记得重启一下 Claude Code，让新 plugin 生效。

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

剩下三个没有写进脚本里，是因为它们的安装命令依赖一些细节（包名、MCP 传输方式、凭证等），这些东西改得比较勤，硬写进脚本里不太安全。对应的 repo README 里都有最新的装法。

### 用另一个模型交叉检查 Claude

同一个模型审自己的代码是有盲区的。两个不同的模型，则很少有*一样*的盲区。[**codex-plugin-cc**](https://github.com/openai/codex-plugin-cc) 这个 plugin 让您不用离开 Claude Code 就能把一段 diff 交给 OpenAI 的 Codex 做第二意见。做预测时，您本来就更信任 ensemble，而不是任何一个单独的模型。同样的思路完全可以套用到代码审查上。要抓住那种 Claude 很自信地写出来、又很自信地给自己盖章通过的 bug，这差不多是成本最低的办法了。

**什么时候该用。** 任何一个非平凡的 diff，只要它动到了训练代码、eval 代码或者数据管线。不是每一个一行的笔误都需要，但任何一个改动只要可能让它产生一个看起来合理但其实错误的数字，就值得过一遍。研究代码里有一种很让人难受的不对称：发布一个悄无声息的 metric bug 的代价，比花两分钟做一次交叉审查的代价要大得多。尤其是在您启动一次长训练、或者对外发布一组数字之前，一定要用。

**怎么用。** Claude 写出一段 diff 之后，您真正 apply 之前：

> "Send this diff to Codex for review. Ask it to look specifically for: off-by-one errors, silent type coercions, reduction-axis mistakes in tensor ops, and anything that would change the numeric output without raising an error. Come back with a ranked list of concerns and the specific lines each concern points at."

让它给出具体行号的那一句，是整个 prompt 里最关键的一句。一句模糊的看起来还行没有任何价值；而一个具体的提醒，比如第 47 行用了 `dim=0` 但可能应该是 `dim=-1`，才是您真正能照着去动的东西。

**怎么看这份审查结果。** 也别把 Codex 的结论当成 ground truth。它也只是另一个模型，有它自己的盲区。您真正应该关注的是*分歧*：当两个模型标的东西不一样、或者其中一个在为某个选择辩护而另一个在质疑时，那一处就是您应该亲自读一遍代码、然后自己拿主意的地方。重点不是让它们达成一致，而是让两副不同的眼睛去看最需要被看的那几行。

### 让 Claude 自己维护一份开发日志

只要一个项目会跨越很多次会话，就让 Claude 自己在 `docs/dev/` 目录下记笔记，每一件值得留下来的事一份短文件。每份文件里写清楚：症状是什么、您试过什么、最后什么真的 work 了、下次值得记住的那一句话是什么。下面这个 prompt 可以在第一天就把这个习惯建立起来：

> "From now on, whenever we hit a non-trivial bug, debug a surprising behavior, or make a design decision with real tradeoffs, write a short note in `docs/dev/YYYY-MM-DD-<slug>.md` with: problem, root cause, what we tried, what ended up working, and a one-line lesson. Keep each file under a page. Before starting any new task in this repo, re-read the most recent notes in `docs/dev/` and tell me if anything there is relevant to what I'm about to ask."

每一份笔记都是一分钟能写完的事，而且之后基本不需要维护。但它们会一点一点堆起来。几十次会话之后，您会发现 Claude 开始在重复一次您已经跑过的调试之前先回去查自己过去的笔记；开始在多个 bug 之间找到共同模式；甚至在您快要踩进一个过去笔记里已经警告过的坑时，主动把您拦下来。不夸张地说，这就是 agent 在 memory 层面的 self-evolving：每一次会话结束时，它在*您这个具体的项目上*都比会话开始时多积累了一点经验。您不需要 fine-tuning，也不需要什么复杂的长程记忆架构。一个 markdown 文件夹加上一条让 Claude 先去读它的 prompt，就已经把这件事大部分的价值拿到手了。

## 如何在工作过程中写好 prompt

把基础设置做完之后，您终于可以真正开始用 Claude 来做事了。您和 Claude 交互的主要方式就是写 prompt，而好的 prompt 并没有什么玄学。所谓好的 prompt，其实就是您在动键盘之前已经想清楚的所有东西的一个压缩版本。

### Spec -> Plan -> Code

研究者的直觉是先跑一下看看。把论文里的伪代码粘过来，跑一下，看看发生什么。这一招在小改动上是管用的，但是在任何一个稍微正经一点的任务上就崩了。三小时之后，您会发现自己深陷在一个连自己都没同意过的大重构里，而且完全不知道做完了到底长什么样。

最近流行起来的那套纪律其实很简单：在写 plan 之前先写 **spec**，在写 code 之前先写 **plan**。

- **Spec** 回答*是什么*和*为什么*。我们到底在做什么？成功的标准是什么？哪些事情是明确不做的？
- **Plan** 回答*怎么做*。要动哪些文件？按什么顺序？哪里最容易出问题？
- **Code** 只是一份被批准过的 plan 的机械执行。

每一个阶段都是改起来很便宜、跳过代价很贵。对做研究的人来说，这个映射特别自然：spec 就是您提的那个实验，plan 就是 runbook，code 就是实现。顺带还有一个好处：六周之后，当您已经忘了当时为什么那样选的时候，那份写下来的 spec 恰恰是您最想翻出来看的那份实验笔记。

**下次任务就试一下。** 不要在 Claude 里一上来就敲 *implement X*，而是从下面这样开始：

> "Before writing any code, draft a one-page spec for X: what we're building, success criteria, explicit non-goals, and the riskiest assumptions. Ask me questions until you're confident the spec is right."

Spec 被批准之后：

> "Now turn this spec into a step-by-step implementation plan. For each step, list the files you'll touch and what could go wrong. Do not write code yet."

一个任务做下来，您就会感觉到区别了。

### 使用 subagents

Subagent 是一种把独立任务派出去并行做、同时又不把主上下文弄乱的办法。每个 subagent 跑在它自己的对话里，替您把那些杂音（文件内容、grep 输出、失败的尝试）全都吸掉，最后只把一个干净的答案交回来。您可以像看待后台进程那样看待它们：凡是可以甩出去的，就甩出去。

**什么时候该用。** 判断很简单。一个任务可以交给 subagent 的条件是：(a) 它是*独立*的，您不需要中途跟 Claude 来回讨论；(b) 它是*嘈杂*的，在主对话里直接做会灌进来几百行文件内容或日志。下面这些例子可以用来找感觉：

- 把 repo 里所有设置学习率的地方都找出来，给我一张表：文件名、行号、具体的值。
- 读这 12 个候选文件，告诉我哪一个包含我们要找的那个 metric 的定义。
- 跑一遍测试套件，只告诉我哪些测试挂了，每个挂掉的原因用一句话总结。
- 这份 800 行的 YAML 配置实际上控制了什么？按子系统分组总结。
- 审计 `notebooks/` 下所有 `*.ipynb` 文件，列出任何还在从 `src/legacy/` 里 import 的 notebook。

**怎么派一个出去。** 不需要什么特殊语法，您直接说就行。下面这样的 prompt 就能工作：

> "Dispatch a subagent to answer this: <question>. The subagent should <read these files / run this command / search for this pattern>. Return only the final answer plus file/line references. Do not include intermediate file contents or tool output in the reply."

最后那句话是最关键的一句。不说它的话，subagent 很可能很认真地把它读过的所有东西都贴回到回复里，那样就完全失去了用 subagent 的意义。您得明确告诉它：把噪音吞掉，只把信号递回来。

**一次派好几个。** 真正的加速来自并行。两个或四个 subagent 同时跑，耗时差不多等于一个的耗时：

> "Dispatch four subagents in parallel. Subagent 1: audit `src/` for learning-rate settings. Subagent 2: audit `configs/` for the same. Subagent 3: check `scripts/` for any hardcoded overrides. Subagent 4: check `notebooks/` for ad-hoc experiments. Each returns a short table. Collect all four into one summary when they're done."

**什么时候不要用。** 如果您需要和 Claude 一来一回地讨论一个答案，不管是争论、反驳还是迭代，就让任务留在主对话里。Subagent 是给一次性查询和批量工作用的，不是给对话用的。另外，如果一个任务小到在主对话里三行就能答完，也别去派 subagent，派发本身那点成本根本不值得。

### Meta-Prompting

有时候一个任务最难的部分是该怎么问。Meta-prompting 就是字面意思：用 Claude 来改进您准备发给 Claude 的那条 prompt。听起来有点傻，但您真正试过一次就会发现，您写的第一版经常漏掉了任务实际需要的一半上下文。

有两种可靠的做法。

**1. 批评并重写。** 您手上有一份草稿，让 Claude 先把它拆开来骂一遍，再去跑。

> "Here's the prompt I'm about to run: <paste>. Don't execute it yet. Critique it: what is ambiguous, what context is missing that would change the answer, and what will you most likely hallucinate or get wrong if I run it as-is? Then rewrite it to fix those issues."

**2. 反向引导。** 您有一个模糊的目标，让 Claude 先来面试*您*，把它做好这个任务需要您先回答的问题列出来，然后再动手。

> "I want to investigate why our model underperforms on long-context inputs. Before doing anything, list the five questions you would need answered to run this investigation well: what evidence, what files, what metrics we should agree on upfront. I'll answer them, and then you'll write the prompt for the actual investigation."

**一个真实的例子。** 假设您想跑一组消融实验，但并不确定哪些东西值得消融。直接问 Claude 您应该消融什么，只能换来一份谁都写得出来的通用清单。但如果您先让 Claude 围绕模型、数据和假设来面试您，让它把那几个最能改变它推荐的问题问出来，然后再根据您的回答给您一份带优先级的方案，它换回来的就是一份真正对着您当下情况定制的清单。道理很简单：在 Claude 给答案之前，它已经拿到了给一个有用答案所需要的上下文。您花五分钟回答几个问题的代价，换的是一小时不用跟一份通用清单扯皮的时间。

Meta-prompting 其实就是把 Claude 当成一个队友而不是一个键盘这个思路再往前推一步。既然您是在 onboard 一个新队友，那有时候最有效的事情，恰恰是让这个新队友先反过来面试您。

### Prompt 示例

提升 prompt 能力最快的方法，就是把同一个任务的两条 prompt 并排摆在一起，然后问自己：为什么第二条能 work？不是因为它更长，而是因为写它的人在动键盘之前已经想清楚了更多东西。

#### 示例 1：让 Claude 复现一个 baseline

**糟糕的 prompt：**

> "Help me reproduce the results of this paper."

**为什么糟糕。** *Reproduce* 根本没被定义。哪张表？哪些数字？误差范围是多少？repo 的上下文也没给。没 spec，没 plan，就是在老虎机上拉了一把。Claude 最后要么猜一个错的目标，要么反问您十个本来应该是您自己先写清楚的问题。

**好的 prompt：**

> "Read this repo and draft a `CLAUDE.md` summarizing entry points, run commands, and data assumptions. Then draft a spec for reproducing Table 2 of the paper: hardware, seeds, configs, and success = within 0.3 points of the reported numbers. Push back if anything is ambiguous. Dispatch subagents to audit `src/`, `configs/`, and `scripts/` in parallel."

**为什么好。** 它点名了具体的交付物：先 `CLAUDE.md`，再 spec。它用一个数字定义了成功的标准。它要求 Claude 在不清楚的地方主动反驳，而不是闷头瞎猜。它还通过 subagent 把嘈杂的探索并行化了。说白了，本文前面几乎所有的原则都被压缩进了这一条 prompt 里。

#### 示例 2：让 Claude 调试 NaN loss

**糟糕的 prompt：**

> "My loss is NaN, fix it."

**为什么糟糕。** 没上下文、没证据、也没说您已经排除过什么。它在没诊断之前就要修，于是您最后会拿到三次"修法"尝试，而每一次都只是糊上了一个没验证过的猜测。而且它还会浪费 Claude 的回合，让它去问那些您本来就知道答案的问题。

**好的 prompt：**

> "Symptom: training loss goes to NaN starting at step 3140 (log tail below). Already ruled out: FP16 overflow (we're in FP32), corrupted input batch (inspected step 3139's batch manually). Before proposing any fix, write a short bug spec, observed vs. expected behavior, candidate hypotheses ranked by likelihood, and the cheapest experiment that would falsify each. We'll test them one at a time."

**为什么好。** 它把证据直接递到了 Claude 手里；它关掉了您已经排除过的门，让 Claude 不会去重复您的工作；它要求在修 bug 之前先写一份 bug spec，这就迫使它把所有的假设显式地排个序，而不是靠瞎猜乱补、再试一次这种方法。这正是本节前面讲过的 Spec → Plan → Code 纪律，应用到调 bug 上的版本。

#### 示例 3：让 Claude 分诊四十篇论文

**糟糕的 prompt：**

> "Read these forty papers and tell me which ones are important."

**为什么糟糕。** *Important* 没有定义。对什么而言重要？把四十份 PDF 一份一份顺序处理，会把主上下文灌满原始论文正文，最后留给您一个六周之后您自己都没法重读的无结构大块文本。没有并行，没有结构，也没有相关性判据。

**好的 prompt：**

> "Spawn one subagent per PDF in `papers/`. Each subagent summarizes its paper against this template: (1) problem, (2) method in two sentences, (3) headline result, (4) relevance to our work on <topic>, (5) verdict, must-read / skim / skip. Collect all results into a single markdown table sorted by relevance. Do not include raw paper text in the replies."

**为什么好。** 它通过 subagent 做了并行。输出是有结构的，所以之后您还能按字段过滤。对您具体研究工作相关性的那一栏，把本来模糊的重要性变成了一个具体的东西。最后那条压噪声的子句又保证了主上下文在 subagent 回报结果之后还能继续用。

#### 示例 4：让 Claude 计算一个自定义 metric

**糟糕的 prompt：**

> "Compute <custom metric> on the eval set and report the number."

**为什么糟糕。** 您会拿到一个数字。这个数字看起来很合理。您把它引用了。然后某一天，某个人发现 metric 的实现里有一个 off-by-one，而这个看起来合理已经让您付出了一轮修订的代价，甚至更糟，一份 corrigendum 的代价。您没验证过的数字，就是您将来要后悔引用的数字。

**好的 prompt：**

> "Before running <metric> on the real dataset, construct a tiny synthetic example with a known answer I can compute by hand. Run the metric on the synthetic example and show me the output. Only after I confirm it matches the hand-computed answer, run it on the real data."

**为什么好。** 它把先验证再信任、尤其是对数字这条原则彻底落到了实处。一个合成的、答案已知的测试用例几乎不花什么成本；而引用一个错的 metric 的代价，并不便宜。

## 结语

Claude Code 真正的价值不是快，是注意力。每一次您干净利落地把一件事甩给它，都是您把一份注意力拿了回来。您可以把这份注意力花在研究里那些真正需要人的部分：品味、判断、下一个该问什么问题、那个对不上的结果背后大概率藏着的那件事。

把工具用对之后，这个转变是显而易见的：您的一天不再是关于打字的，而是关于做决定的。这，就是聪明地使用 Claude 的意思。

## Citation

如果这篇文章对您的研究工作有帮助，欢迎引用：

```bibtex
@misc{wang2026claudesmart,
  author       = {Ziyi Wang},
  title        = {How to Use Claude the Smart Way},
  year         = {2026},
  howpublished = {\url{https://github.com/ivowang/how-to-use-claude-the-smart-way}}
}
```
