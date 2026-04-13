# 使用 Claude 的智慧

[English](README.md)

大多数人初次接触 Claude Code，会把它归类为"一个更高级的自动补全工具"。正是这种定位让他们拿到的结果平平无奇。Claude Code 不是更快的键盘，它是一个你可以放心委派任务的协作者。这个区别对研究者尤其重要，因为你的瓶颈从来就不是打字速度，而是注意力：那份决定"该跑哪个实验、哪个 bug 是真的、哪篇论文值得读"的稀缺预算。用对了，Claude Code 会把这份注意力还给你;用错了，它只会生成更多需要你审查的东西。

## 在开始工作前如何配置好 Claude Code

有几个功能值得你在第一天就学会，每一个都会在第一次用到时就回本。

### 写一份 CLAUDE.md

这是一份放在 repo 根目录下的纯文本文件，Claude 在每次会话开始时都会自动读取它。把一个新协作者需要知道的东西都写进去:这个项目是什么、怎么跑、哪些目录重要、哪些坑已经有人踩过。花十分钟写这份文件，能省下后面几个小时反复解释的时间，而且每一次会话都在复利地收获价值。

下面是一份起始模板，你可以粘贴到自己的 repo 里:

```markdown
# Project: <name>

## What this is
One paragraph: the research question， the approach， and what counts as success.

## Local environment
- **GPUs:** 8× A800 40GB. Use whichever are idle — check with `nvidia-smi` before launching a run.
- **Conda env:** `clsea` is already set up; `conda activate clsea`. Do not create a new env.
- **Versions:** Python 3.11， CUDA 12.1， torch 2.3.1 (pinned — see Gotchas).

## How to run things
- Train: `python train.py --config configs/base.yaml`
- Eval:  `python eval.py --ckpt checkpoints/latest.pt`

## Where things live
- `configs/`   — experiment configs; `base.yaml` is the canonical starting point
- `data/`      — raw in `data/raw/`， processed in `data/processed/v2` (v1 is stale)
- `src/`       — model and training code; read this， don't touch `legacy/`
- `scripts/`   — one-off analysis; not source of truth
- `notebooks/` — exploration only; never import from these

## Network
We're behind the Great Firewall. If `pip`， `git`， HuggingFace， or any OpenAI/Google endpoint hangs， run `clashon` to enable the proxy and retry. A hang is almost always a network issue， not a bug — don't debug the wrong problem.

## LLM API access
API keys for Qwen / GPT / Gemini / Claude live in `.env` at the repo root. Load them with `python-dotenv` or `source .env`. Never hardcode a key in a script， and never commit `.env`.

## Git
Remote is `github.com/ivowang/how-to-use-claude-the-smart-way`. Commit and push regularly; prefer small， self-contained commits over a single end-of-day dump.

## Gotchas
- Eval silently skips NaN-loss batches; check `logs/skipped.txt` after every run
- Do NOT upgrade torch past 2.3.1 — breaks the custom CUDA kernel in `src/ops/`
- Seeds are set in `configs/base.yaml`， not in the training script
- W&B project is `myproject-dev`; `myproject` is reserved for paper runs
```

认真把每一栏填满。

### 使用 subagents

把独立的任务派发给独立的 agent，让它们并行地工作而不污染你的主上下文。Subagent 在自己的对话里运行，替你吸收噪音(文件内容、grep 输出、失败的尝试)，最后只把一个干净的答案交到你手里。研究者应该像看待后台进程一样看待 subagent:只要是独立的任务，就派发出去。

**什么时候该用 subagent。** 判断很简单。如果一个任务同时满足:(a) *独立*，你不需要在中途和 Claude 来回讨论;(b) *嘈杂*，在主对话里直接做会灌进去几百行文件内容或日志，那它就应该放到 subagent 里。照着下面几个例子去找感觉:

- "把 repo 里所有设置学习率的地方都找出来，给我一张表:文件名、行号、具体的值。"
- "读下面这 12 个候选文件，告诉我哪一个包含我们要找的那个 metric 的定义。"
- "跑一遍测试套件，只告诉我哪些测试挂了，每个挂掉的原因用一句话总结。"
- "这份 800 行的 YAML 配置实际上控制了什么?按子系统分组总结。"
- "审计 `notebooks/` 下所有 `*.ipynb` 文件，列出任何还在从 `src/legacy/` 里 import 的 notebook。"

**怎么派发一个 subagent。** 你不需要什么特殊语法，你只是把它说出来。下面这个 prompt 就能工作:

> "Dispatch a subagent to answer this: <question>. The subagent should <read these files / run this command / search for this pattern>. Return only the final answer plus file/line references. Do not include intermediate file contents or tool output in the reply."

最后那句话是关键。没有它的话，subagent 可能会乖乖地把它读过的所有东西都贴回回复里，那样就完全失去了用 subagent 的意义。你必须显式地告诉它:替我把噪音吞掉，只把信号交回来。

**同时派发多个。** 真正的倍增器是并行。两个或四个 subagent 同时跑，耗时大约等于一个:

> "Dispatch four subagents in parallel. Subagent 1: audit `src/` for learning-rate settings. Subagent 2: audit `configs/` for the same. Subagent 3: check `scripts/` for any hardcoded overrides. Subagent 4: check `notebooks/` for ad-hoc experiments. Each returns a short table. Collect all four into one summary when they're done."

**什么时候不要用 subagent。** 如果你需要和 Claude 反复讨论一个答案，如争论、反驳、迭代，就留在主对话里。Subagent 是为一次性查询和批量工作准备的，不是为对话准备的。另外，如果一个任务小到三行就能答完，也别派发 subagent，那点派发成本不值得。

### 用 skills 和 plugins 扩展 Claude

Skills 和 plugins 是 Claude 可以按需调用的可复用能力。不用每次会话都重新解释你想要的 TDD、调试或文献检索工作流是什么样子，装一次 skill，以后每次会话它都在。这是整个生态中复利效应最明显的地方，值得你在这里投入真正的时间。

**你最好别忽略的技能**

- [**superpowers**](https://github.com/obra/superpowers) 它打包了一组久经考验的 skills:brainstorming、writing plans、systematic debugging、TDD、verification-before-completion。一分钟以内安装完成，你的基线质量会明显提升，而且你不需要额外做任何事情。

**官方 Anthropic plugins**([anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official))。一套精心挑选的一方 plugin。和研究工作最相关的几个是:

- [**feature-dev**](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/feature-dev)，一个结构化的 spec → plan → implement → verify 工作流，打包成了 plugin。和上面提到的纪律直接呼应。
- [**code-simplifier**](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/code-simplifier)，审查改动过的代码的复用性、质量和效率，然后修掉发现的问题。能防止原型代码在合作者看到之前就烂成面条。
- [**claude-md-management**](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/claude-md-management)，帮你创建、审计和维护 CLAUDE.md。一份干净的 CLAUDE.md 会让之后每一次会话的质量都变好，而这个 plugin 帮你解决"我回头再更新"这个借口。
- [**skill-creator**](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/skill-creator)，写新 skill 用的元 skill。把一个可复用的研究工作流(你偏好的画图风格、你的数据加载样板代码、你的 eval runner)捕捉成你自己的 skill，这是最快的路径。
- [**ralph-loop**](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/ralph-loop)，让 Claude Code 递归地、自主地循环运行，直到某个 spec 被满足为止。专为无人值守的通宵迭代设计:配置扫参、不稳定测试矩阵、睡觉的时候啃完 TODO 列表。

**浏览器自动化和视觉验证。**

- [**playwright-skill**](https://github.com/lackeyjb/playwright-skill)，让 Claude 通过 Playwright 驱动一个真实的浏览器，跑 E2E 测试并验证视觉输出。如果你在搭 web 端的 demo、交互 dashboard 或者 paper artifact 站点，可以让 Claude 真的端到端地确认"东西确实能 work"，而不是只确认"代码能编译"。

**文献和论文。**

- [**paper-search-mcp**](https://github.com/openags/paper-search-mcp)，一个 MCP server，让 Claude 直接访问 arXiv、PubMed、bioRxiv、medRxiv 和 Google Scholar。装了它之后，你的文献分诊工作流不再是"手动把 PDF 丢进一个文件夹"，而是"让 Claude 自己去拿"。

**索引目录。** 这个生态变化很快，任何写在 README 里的列表都会过期。与其硬编码一份清单，不如把规范的索引加进书签，每隔几周浏览一次:

- [**awesome-claude-code**](https://github.com/hesreallyhim/awesome-claude-code)，社区维护的 skills、hooks、slash-commands、plugins 和 agent orchestrators 的索引。每隔一段时间花五分钟逛一下，你会抢在生态"发现你"之前先发现生态。

**快速安装脚本。** 把下面这段脚本粘贴到终端里，一次装完上面推荐的所有 plugin。需要你已经安装了 Claude Code CLI(`claude --version` 能跑通就行)。脚本可以重复运行;装完之后重启一下 Claude Code 让新 plugin 生效。

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

剩下三个项目留作手动步骤，是因为它们的安装命令依赖的细节(包名、MCP 传输、凭证)变动得比较频繁，硬编码在这里不安全，对应的 repo 里有最新的指引，每一个都只需要花一分钟。

### 用另一个模型交叉检查 Claude

一个模型审查自己的代码会有盲区。两个不同的模型则很少有同样的盲区。[**codex-plugin-cc**](https://github.com/openai/codex-plugin-cc) 这个 plugin 让你可以在 Claude Code 里直接把一段 diff 交给 OpenAI 的 Codex 做第二意见。研究者在做预测时本来就信任集成模型多过单个模型，同样的逻辑也适用于代码审查。这是抓住"Claude 自信地写出来、又自信地认可了"的那种 bug 最便宜的方法。

**什么时候该用它。** 任何一个 non-trivial 的 diff， 只要它动到了训练代码、eval 代码或数据管线。不是每一个一行的笔误修正都需要，但任何一个"一个微妙的 bug 会产生看起来合理但其实错误的数字"的改动都需要。研究代码有一个令人痛苦的不对称:发布一个悄悄的 metric bug 的代价，比花两分钟做一次交叉审查的代价大了几个数量级。尤其要在你启动一次长训练、或者发布一组数字之前用它。

**怎么用它。** Claude 提出一段 diff 之后、你真正 apply 它之前:

> "Send this diff to Codex for review. Ask it to look specifically for: off-by-one errors， silent type coercions， reduction-axis mistakes in tensor ops， and anything that would change the numeric output without raising an error. Come back with a ranked list of concerns and the specific lines each concern points at."

最后那句 "specific lines" 很关键，一个模糊的"看起来还行"是没用的;一个具体的"第 47 行用了 `dim=0`，我觉得你应该用 `dim=-1`"才是可行动的。

**怎么处理审查结果。** 不要把 Codex 的结论也当成 ground truth，它也只是另一个模型，有它自己的盲区。真正的信号是*分歧*:当两个模型标记了不同的东西、或者一个为某个选择辩护而另一个质疑它时，那一处就是你应该亲自读代码并自己决定的地方。重点不是达成共识，重点是让两副不同的眼睛看向最需要被看的那几行代码。

## 如何在工作过程中写好 prompt

前面讲的一切，思维模式的转变、spec-first 的纪律、subagent、验证，最终都会汇聚到你真正敲下去的那段 prompt 里。一个好的 prompt 不是什么魔法措辞。它是你在碰键盘之前就已经想清楚的一切东西的压缩形式。

### Meta-Prompting

有时候一个任务最难的部分是"该怎么问"。**Meta-prompting** 就是它字面上的意思:用 Claude 来改进你准备发给 Claude 的那段 prompt。听起来有点傻，但你真正试一次之后会发现，你的初稿常常缺了一半任务实际需要的上下文。

有两种可靠的形式:

**1. 批评并重写。** 你有一份草稿，让 Claude 先把它拆开看看问题，再去跑。

> "Here's the prompt I'm about to run: <paste>. Don't execute it yet. Critique it: what is ambiguous， what context is missing that would change the answer， and what will you most likely hallucinate or get wrong if I run it as-is? Then rewrite it to fix those issues."

**2. 反向引导。** 你有一个模糊的目标，让 Claude 先向*你*提问:把它做好这个任务需要你先回答的问题列出来，然后再开始动手。

> "I want to investigate why our model underperforms on long-context inputs. Before doing anything， list the five questions you would need answered to run this investigation well — what evidence， what files， what metrics we should agree on upfront. I'll answer them， and then you'll write the prompt for the actual investigation."

**一个实际案例。** 假设你想跑一组消融实验，但不确定哪些因素值得消融。天真版的 prompt，"我应该消融什么?"，只会换来一份谁都写得出来的通用检查单。Meta 版的 prompt，"在提出消融方案之前，先问我五个关于模型、数据和假设的问题，这些问题的答案最能改变你的推荐;然后根据我的回答，给我一份带优先级的方案"，能换来一份针对你具体情况量身定制的列表，因为 Claude 现在拿到了它给出有用答案所需要的上下文。花五分钟回答那几个问题的代价，省下的是一小时和一份通用列表拉扯的时间。

Meta-prompting 其实只是"思维模式转变"再往前推一步:既然你是在 onboard 一个新队友，那有时候最高效的事情是让这位队友先面试你。

### Prompt 示例

提升 prompt 能力最快的方法，是把同一个任务的两个 prompt 并排放在一起，然后问自己:为什么第二个 prompt 能 work?差别不在长度。差别在于作者在下笔前已经决定了什么。

#### 示例 1:让 Claude 复现一个 baseline

**糟糕的 prompt:**

> "Help me reproduce the results of this paper."

**为什么糟糕。** "Reproduce" 没有定义，哪张表?哪些数字?在什么误差范围内?没有 repo 的上下文，没有 spec，没有 plan，就是一次老虎机式的抽奖。Claude 要么猜错目标，要么反问你十个本来应该是你先写清楚的问题。

**好的 prompt:**

> "Read this repo and draft a `CLAUDE.md` summarizing entry points， run commands， and data assumptions. Then draft a spec for reproducing Table 2 of the paper: hardware， seeds， configs， and success = within 0.3 points of the reported numbers. Push back if anything is ambiguous. Dispatch subagents to audit `src/`， `configs/`， and `scripts/` in parallel."

**为什么好。** 它点名了具体的交付物(先 CLAUDE.md，再 spec)，用数字定义了成功的标准，主动邀请反驳以防 Claude 闷头瞎猜，并通过 subagent 把嘈杂的探索并行化了。本文前面所有的原则都被压缩进了这一条 prompt。

#### 示例 2:让 Claude 调试 NaN loss

**糟糕的 prompt:**

> "My loss is NaN， fix it."

**为什么糟糕。** 没有上下文、没有证据、没有排除过什么。它在诊断之前就要"修"，这样你最后会拿到三个"修法"，每一个都只是糊上了一个没验证过的猜测。而且它还浪费了 Claude 的回合去问那些你本来就知道答案的问题。

**好的 prompt:**

> "Symptom: training loss goes to NaN starting at step 3140 (log tail below). Already ruled out: FP16 overflow (we're in FP32)， corrupted input batch (inspected step 3139's batch manually). Before proposing any fix， write a short bug spec — observed vs. expected behavior， candidate hypotheses ranked by likelihood， and the cheapest experiment that would falsify each. We'll test them one at a time."

**为什么好。** 它把证据直接递给 Claude，关掉了你已经确认过的门，让它不用重复你的工作;而且要求在修 bug 之前先写一份 bug spec，这迫使它把假设显式地排个序，而不是靠猜-补-测。这正是前面讲过的 Spec → Plan → Code 纪律应用在调 bug 上的版本。

#### 示例 3:让 Claude 分诊四十篇论文

**糟糕的 prompt:**

> "Read these forty papers and tell me which ones are important."

**为什么糟糕。** "Important" 没有定义，对什么而言重要?顺序处理四十份 PDF 会把你的主上下文灌满原始论文正文，最后留给你一个六周之后自己都没法重读的无结构大块文本。没有并行、没有结构、没有相关性判据。

**好的 prompt:**

> "Spawn one subagent per PDF in `papers/`. Each subagent summarizes its paper against this template: (1) problem， (2) method in two sentences， (3) headline result， (4) relevance to our work on <topic>， (5) verdict — must-read / skim / skip. Collect all results into a single markdown table sorted by relevance. Do not include raw paper text in the replies."

**为什么好。** 通过 subagent 并行、输出结构化(之后可以直接过滤)，给了一个显式的相关性判据把"重要"变成具体的东西，并且有一条压噪声的子句把主上下文保持在可用状态。

#### 示例 4:让 Claude 计算一个自定义 metric

**糟糕的 prompt:**

> "Compute <custom metric> on the eval set and report the number."

**为什么糟糕。** 你会拿到一个数字。它看起来很合理。你把它引用了。最终有人会发现 metric 实现里有一个 off-by-one，而这个"看起来合理"已经让你花掉了一个修订轮次，或者一份 corrigendum。没验证的数字就是你将来要后悔引用的数字。

**好的 prompt:**

> "Before running <metric> on the real dataset， construct a tiny synthetic example with a known answer I can compute by hand. Run the metric on the synthetic example and show me the output. Only after I confirm it matches the hand-computed answer， run it on the real data."

**为什么好。** 它把思维模式转变里的第三条引申，"先验证再信任，特别是对数字"，具体落地了。一个合成的、答案已知的测试用例代价很低;引用了一个错的 metric 的代价不低。

## 处理复杂任务:Spec -> Plan -> Code

研究者的直觉是"先跑一下看看"。把论文里的伪代码粘过来，跑起来，看看结果。这招在小改动上管用，但在任何一个稍微正经的任务上都会崩，三小时之后你会发现自己深陷在一个你从未同意过的大重构里，而且完全不知道"做完了"长什么样。

现在流行起来的纪律很简单:在 plan 之前先写 **spec**，在 code 之前先写 **plan**。

- **Spec** 回答*是什么*和*为什么*:我们要做什么，成功的标准是什么，明确不做什么。
- **Plan** 回答*怎么做*:要改哪些文件、按什么顺序、哪里可能坏。
- **Code** 是一份经过批准的 plan 的机械执行。

每一个阶段都是改起来便宜、跳过代价昂贵。对做研究的人来说，这个映射非常自然:spec 就是你在提议的那个实验，plan 就是 runbook，code 就是实现。另外还有一个好处:六周之后当你已经忘了当初为什么那样选时，一份写下来的 spec 恰恰是你最想回头查的实验笔记。

**下次任务就试一下这个。** 不要在 Claude 里上来就敲 "implement X"，试试这样开头:

> "Before writing any code， draft a one-page spec for X: what we're building， success criteria， explicit non-goals， and the riskiest assumptions. Ask me questions until you're confident the spec is right."

Spec 被批准之后，继续:

> "Now turn this spec into a step-by-step implementation plan. For each step， list the files you'll touch and what could go wrong. Do not write code yet."

做完一个任务你就会感觉到区别。

## 结语

Claude Code 真正的解锁效果不是速度，是注意力。每一个被你干净利落地委派出去的任务，都是被你拿回来的注意力，你可以把这份注意力花在研究里那些真正需要人的部分:品味、判断、"下一个该问什么问题"、那个不合模型的结果背后很可能藏着的东西。

把工具用对了，这个转变是明显的:你的一天不再是关于打字的，而是关于做决定的。这就是聪明地使用 Claude 的方式。

## Citation

如果这篇文章对你的研究工作有帮助，欢迎引用:

```bibtex
@misc{wang2026claudesmart，
  author       = {Ziyi Wang}，
  title        = {How to Use Claude the Smart Way: A Short Guide for AI Researchers}，
  year         = {2026}，
  howpublished = {\url{https://github.com/ivowang/how-to-use-claude-the-smart-way}}
}
```
