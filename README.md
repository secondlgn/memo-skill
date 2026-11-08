# memo — 工作区注意力本本

一个给 Agent 自我调节注意力的**滚动备忘录**，用来对抗两种典型毛病：
**健忘**（把该盯的事忘了）和 **钻牛角尖**（在某细节上钻太深而忽略主线）。

- 它**不是记忆仓库**，而是「现在该盯着什么」的、有预算的取舍结果。
- 数据**跟随工作目录**，多个 Agent（pi / Codex / …）共用同一份索引，靠 `agent` 字段区分。
- Agent 自己维护排序，用户**扫一眼文件夹**即可诊断注意力是否偏离。

## 目录结构

```
memo-skill/
  SKILL.md            # 给 Agent 的完整规则（何时记、如何估值、如何消解冲突）
  scripts/memo        # 单命令：insert / update / drop / view / playbook / route
  playbooks/          # 场景化种子备忘（issue-to-ship, need-to-design …）
  prompts/memo.md     # /memo 斜杠指令模板（拷到 ~/.pi/agent/prompts/ 启用）
  memo/.gitkeep       # 运行时数据目录（index.json + 条目 .md + history/），不随仓库分发
```

> `memo/` 里是各工作区自己的运行时数据，**不提交**（见 `.gitignore`）。仓库只装「机制」。

## 文件形态

每条 memo 一个文件：`{NNN}-{描述}.md`（NNN 固定 3 位序号，描述 5~10 字）。
重要性绝对值、agentID、时间戳等结构化字段都存索引 `memo/index.json`，**文件名只留人眼可读的序号+描述**。序号是每次 insert/update/drop 后由脚本重排重命名得到的派生值。

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
```

命令详情见 `SKILL.md`。Agent 用起来只需传语义参数，重排、重命名、写索引全部由脚本自动完成。

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

## license / 依赖

- 纯 Python 标准库，无第三方依赖。
- 可用 `MEMO_DIR` 环境变量指定任意工作区数据目录。
- `GITHUB_REPO` 环境变量可覆盖提 issue 的仓库地址（默认占位符）。