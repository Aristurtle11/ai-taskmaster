# 简单三步实现 AI 无间断自动开发

前几天转发了一条宝玉老师很久以前的一个帖子，内容大概是他做了一个 ai 监工系统，可以无人值守的情况下独立工作。我看了这个帖子之后深有启发，恰好手上又有个小 task ，刚好拿来试一下。

虽然最终 ai 生成的代码并不理想，但是这毕竟是一次有价值的尝试，于是我就给大家分享一下这次的工作经验。

## Prompts Scaffolding

要让 AI 工作，无论如何都是不可能离开人的指令的。让 AI 做到无人值守，其本质上就是把人应该下的指令全部在开始就全部说完，然后再引导 AI 每次都自己去查询前述的指令。

但是需要解决几个问题：

1. 开一个 codex session 完成所有任务肯定会爆上下文，所以需要周期性退出 codex 并开启新的 session。
2. 退出后新开的 codex 肯定会丢失上下文，所以要引导 AI 去阅读文档和代码以补全上下文。
3. 要记录每次 AI 工作完成后，工程所处的状态，以便新的 session 开启后，能够让 codex 知道下一步应该做什么。

因此我们就需要一个所谓的 Prompts Scaffolding ( 提示词脚手架 )，用这套提示词文件去引导 AI 工作。这套脚手架我也已经开源放在了 Github 上：https://github.com/Aristurtle11/ai-taskmaster


脚手架的结构如下：

```shell
.
├── AGENTS.md
├── codex-monitor.sh
├── docs
│   ├── gemini_api
│   ├── gemini_prompts.md
│   ├── MVP.md
│   ├── TDD.md
│   └── wechat_official
└── PROGRESS.md

3 directories, 6 files
```

其中 `docs/gemini_api`和`docs/wechat_official`是两个文件夹，存放的是一些接口调用指南，为 AI 提供正确的接口使用方法，避免它们乱编。

`gemini_prompts.md`是唯一我自己写的文档，这个文档给 Gemini 的 Prompts，让它来写`MVP.md`，`TDD.md`，`AGENTS.md`，`PROGRESS.md`。

`MVP.md` 描述了软件的用途是什么，要实现什么功能，以及有什么性能或者技术路线上的约束。这个就是为了让 AI 对于产品有个宏观的理解。

`TDD.md`就是技术设计文档。将软件架构，还有能想到的接口都详细写出来，以便 AI 开发时有一个标准参照。

`PROGRESS.md`文档将整个工程切分成最小可行步骤，指示 AI 当前要做什么工作，并且记录哪些工作已经完成，哪些还没有完成。

`AGENTS.md`就是 codex 每次开始工作时阅读的第一个文件，我就是通过它来引导 codex 去阅读其他文档来补充 context 的。

`codex-monitor.sh`是驱动 AI 不停运动的引擎。

## 构建

想让 AI 的工作效果越好，那么就要越用心设计 Prompts Scaffolding。必须把心态从一个开发者转变成一个设计者、管理者，要把 AI 工作中可能需要的东西，遇到的问题都想到，提前把这些东西都准备好。

我的这个项目是来源于我的舅舅，他在佛州做房地产生意，但是客户主要是华人，所以他需要运营一个微信公众号，但是由于精力有限，所以让我给他写一个软件，让他把他在英文网站上看到的文章一键翻译排版发布到微信公众号上。

所以根据这个需求，我写了第一个文档 `docs/gemini_prompts.md`。

然后我把这个文档送入 Gemini，等了一小会就有了 `docs/MVP.md`, `docs/TDD.md`, `PROGRESS.md`。

**注意**这里一定要给 Gemini 强调，这些文档要写给 AI developer，所以引导 AI developer 去指定路径阅读相关文档。

### 关键文件：`codex-monitor.sh`

该文件就是宝玉老师在帖子中的监工脚本，内容如下：

```shell
#!/usr/bin/env bash
#
# codex_task_monitor.sh — minimal scheduler that runs
# `export TERM=xterm && codex exec "continue to next task" --full-auto`
# every five minutes and reports each run's outcome.

set -euo pipefail

cmd=(codex exec "continue to next task" --full-auto)

while true; do
  echo "[codex-monitor] starting run at $(date -u '+%Y-%m-%dT%H:%M:%SZ')"

  if TERM=xterm "${cmd[@]}"; then
    echo "[codex-monitor] codex exec completed successfully (exit 0)."
  else
    exit_code=$?
    echo "[codex-monitor] codex exec exited with errors (exit $exit_code)."
  fi

  echo "[codex-monitor] waiting five minutes before the next run..."
  sleep 300
done
```

总体上这个文件就是驱动 codex 不停工作的引擎。

### 关键文件：`PROGRESS.md`

文件`PROGRESS.md`之所以关键，是因为它不仅为 AI 的工作做出了详细的规划，而且还是一个状态机可以记录 AI 当前工作到了哪一步，下一步又应该是什么。

该文件引导 codex 每次唤醒时去找最上方的空标记任务，然后开始工作前在任务前加一个 [>] 标记，完成后加一个 [x] 标记。

并且告诉 codex 每次完成一项任务后都要提交 git commit 。

### 关键文件：`AGENTS.md`

codex 默认启动后先读该文件，所以该文件就成为了引导 codex 阅读其他文档的桥梁：

```
AGENTS.md
    ↓
docs/TDD.md
docs/MVP.md
PROGRESS.md
    ↓
docs/gemini_api
docs/wechat_official
```

## END

当这一切都准备好之后，你就可以直接在命令行中输入 ./codex-monitor.sh， 迎接最激动人心的时刻吧，一个原始的 AI 劳工诞生了。

大家如果想体验一下具体什么感受的话，可以把我的这个工程下载下来试一下。虽然说这套脚手架可以24小时无间断驱动 codex 工作，但是实际上基本上两个小时多就运行完了，毕竟我这个需求还是一个小需求。

希望大佬多多分享自己的 AI 劳工驯养经验，为我这个简陋的方案提点改进的建议。