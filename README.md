# memo — 工作区注意力本本

一个给 Agent 自我调节注意力的**滚动备忘录**，用来对抗两种典型毛病：
**健忘**（把该盯的事忘了）和 **钻牛角尖**（在某细节上钻太深而忽略主线）。

**注意力优先（attention-first）：** 人和 AI 的注意力都有限。memo 把最要紧的顶到当下（importance 排序），让「不紧急但重要」的不丢（留档可回看）——既不在琐碎上耗散，也不遗忘该记住的事。

- 它**不是记忆仓库**，而是「现在该盯着什么」的、有预算的取舍结果。
- 数据**跟随工作目录**，多个 Agent（pi / Codex / …）共用同一份索引，靠 `agent` 字段区分。
- Agent 自己维护排序，用户**扫一眼文件夹**即可诊断注意力是否偏离。

## 目录结构

```
memo-skill/
  SKILL.md            # 给 Agent 的完整规则（何时记、如何估值、如何消解冲突）
  scripts/memo        # 单命令：insert / update / drop / view / playbook / review / tidy / daydream
  playbooks/          # 场景化种子备忘（issue-to-ship, need-to-design …）
  prompts/memo.md     # /memo 斜杠指令模板（拷到 ~/.pi/agent/prompts/ 启用）
  memo/.gitkeep       # 运行时数据目录（index.json + 条目 .md + history.ndjson），不随仓库分发
```

> `memo/` 里是各工作区自己的运行时数据，**不提交**（见 `.gitignore`）。仓库只装「机制」。

## 文件形态

每条 memo 一个文件：`{NNN}-{描述}.md`（NNN 固定 3 位序号，描述 5~10 字）。
重要性绝对值、agentID、时间戳等结构化字段都存索引 `memo/index.json`，**文件名只留人眼可读的序号+描述**。序号是每次 insert/update/drop 后由脚本重排重命名得到的派生值。

滚动历史是 `memo/history.ndjson`（**只追加 append-only**，每行一条事件，永不覆盖）。每次操作还记 `delta`（对 memo 的重要性影响量：新重要性 / 升降差 / -重要性），用 `memo review` 可按影响量找出「什么造就了重大改变」。多进程并发写由哨兵锁 `.memo.lock` 串行化，跨平台可靠。

### 要事优先（待办也塞进同一张清单）

memo 只按重要性排一列：**已经发生的、和接下来要做的，混在一起凭 importance 排**。「要做的事」不是 todo 系统，而是用法约定——把下一步/待办**直接 `insert` 一条普通 memo**，按紧迫性给 importance，它自然插到该在的位置（要事顶前）；做了之后 `update` 标记或 `drop` 收走。整个计划/下一步 = 一条高 importance、会自己 `update` 的 memo，新版本覆盖旧版本、旧版进历史可复盘。零新字段、零新机制。

## 快速上手

```
# 插入
scripts/memo insert --desc "锁定技术栈" --importance 0.92 --agent <my-id> --note "已确认"
# 更新（自动重排重命名）
scripts/memo update --id <id> --importance 0.75
# 查看个人 top N
scripts/memo view --scope agent --top 10
# 查看滚动日志（全部历史 / 被淘汰的记录）
scripts/memo log --all
# 回顾重大改变（按影响量 |delta| 找历史中的关键事件）
scripts/memo review

# 总结清理：聚类 + 归并/重排/清理候选（供 agent 判断后落地）
scripts/memo tidy
# 白日梦：从历史随机捞被遗忘的旧线索
scripts/memo daydream
```

命令详情见 `SKILL.md`。Agent 用起来只需传语义参数，重排、重命名、写索引全部由脚本自动完成。

## 梳理类命令：tidy / daydream / review

三个命令都遵循「**脚本采集结构、agent(LLM)做判断**」：脚本不自动改数据，只输出结构化素材，agent 据此决定 insert / update / drop 落地。

- **`tidy`** —— 总结清理。输出按来源归类、低重要性候选、疑似可归并项、最近 drop 的旧目标。归并=保留一个+其余 drop 带 note 合并；重排序/重命名由脚本自动完成。
- **`daydream`〔白日梦〕** —— 从历史随机捞被 drop 的旧线索（`--mix` 混入别的操作），对照当前仍在盯的 top3，当灵感/遗忘补给。
- **`review`〔回顾〕** —— 按对 memo 的重要性影响量 |delta| 从大到小列出重大改变事件。写历史时自动记 delta（insert=+新重要性 / update=+升降差 / drop=-重要性），所以历史里能直接找出「触发重大改变的那几次操作」供回到分析。

## Playbook（场景化种子）与 `/memo`

选定一个任务场景时，把它的种子备忘批量灌入当前工作区（`--dry-run` 先预览）：

```
scripts/memo playbook list
scripts/memo playbook init --name issue-to-ship --dry-run
scripts/memo playbook init --name issue-to-ship
```

`/memo` 斜杠指令（需拷 `prompts/memo.md` 到 `~/.pi/agent/prompts/`）让 LLM 读对话上下文、**自动路由到最匹配的 playbook** 并初始化。

- 输入零参数调用：`/memo`；或带意图线索：`/memo issue #42`。
- 不匹配时不会硬塞：引导你到 GitHub 提 issue（`memo playbook route` 打印入口）。
- 可逆操作大胆做；同一操作重试 **3 次仍失败即通知你**。

### 提新场景

在 `playbooks/<slug>.json` 加一个 `seeds` 列表即可被 `playbook list/init` 识别。也欢迎到仓库提 issue（`new playbook: <意图>`）。

## 使用反馈：让 skill 越用越大

用的时候发现命令别扭、报错、哪个场景想复用却没 playbook，都欢迎到仓库提 GitHub issue——`https://github.com/secondlgn/memo-skill/issues`。一次一条，写明在哪个命令/场景、预期与实际的差别即可。所有人的反馈汇聚成改进议题，skill 据此演化，成为更多 AI 愿意用的理由。（脚本零改动：反馈是对 Agent 的用法约定，不是后台自动上传。）

## license / 依赖

- 纯 Python 标准库，无第三方依赖。
- 可用 `MEMO_DIR` 环境变量指定任意工作区数据目录。
- `MEMO_GITHUB_REPO` 环境变量可覆盖提 issue 的仓库地址（默认指向 `secondlgn/memo-skill`）。