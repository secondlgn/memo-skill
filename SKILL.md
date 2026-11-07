# memo — 工作区注意力本本（Agent 自维护）

一个给 Agent 自我调节注意力的滚动备忘录工具，可被任何 Agent（pi、Codex 或其他）复用。
它是**Agent 自己给自己看的认知辅助**，用户随时可以扫一眼工作区文件夹检查注意力是否偏离。

## 设计理念

- **这不是记忆仓库/流水账**。**用到才记**，过一道「值不值得记」的判断题才动笔；鸡毛蒜皮、稳定常识不记。
- **跟随工作目录，不跟随 skill 本身**。memo 数据放在对话的 working folder 下，多个 Agent 共用同一个 `memo/` 文件夹和索引，靠 `agent` 字段区分归属。
- **人扫一眼文件夹（按重要性排序的 `NNN-描述.md`）就能看出：谁重要、次序如何、注意力是否偏了。**

## 记什么（三类内容，跟着当前工作区走）

1. **目标 / playbook** —— 确认过、并被一再强调的偏好与事实（完成标准、主线目标）。
2. **死路 / 面包屑** —— 执行中发现的不可行路径，让我知道自己停在哪个岔口、不重走。
3. **可编辑项** —— 原有条目确实变了，就改掉，而不是堆一条新的。

## 权重 / 重要性怎么定

- **默认由维护的 Agent 当场估**初值。
- **用户明确强调「这个很重要」→ 按用户的来。**
- **拿不准两条谁更重 → 问用户，让用户定高低。**
- 顺序是派生值：脚本按 `(importance DESC)` 重排，重写 `NNN` 序号并重命名文件。**重要性绝对值和 agentID 在索引 `index.json` 里，文件名只留视觉可读的序号 + 描述。**

## 一致性（single source of truth）

- **一个事实只有一条记录、一个权重、一个归属。**
- **冲突必须解决，不并存。** 同一事实出现两个不同取值，必须修改旧条目而不是新增；矛盾即触发 edit 或询问用户，绝不缓存对立的旧值。
- 索引 `memo/index.json` 是唯一真相源，文件名的序号只是它的展示快照。

## 使用命令

所有流程经过 `scripts/memo` 单命令（自动重排、重命名、写索引），Agent 不手搓机械操作：

```
# 插入一条（含重要性初值、来源 agent、一句话说明）
scripts/memo insert --desc "锁定技术栈" --importance 0.92 --agent <id> --note "已确认，优先遵守"
# 更新一条（改重要性/描述 → 自动重排并重命名）
scripts/memo update --id <id> --importance 0.75 --desc "改稿先过 AGENTS.md"
# 移除一条
scripts/memo drop --id <id>
# 读取：个人 top N 或工作区 top N
scripts/memo view --scope agent --top 10          # 我自己的
scripts/memo view --scope workplace --top 10      # 整个工作区共同的
scripts/memo view --scope workplace --all          # 全部
# 查看滚动/审计日志（曾经有过的所有条目、改动、被淘汰的记录）
scripts/memo log --all                             # 全部历史
scripts/memo log --agent <id> --top 20             # 某 Agent 的历史
```

> 工作区默认取当前目录下的 `memo/`；可用环境变量 `MEMO_DIR` 指向任意工作区。
> Agent 来源 `--agent` 取自己的标识（如环境变量里的 agent id）。

## 更新会话

- **每回合结束或每发现「这值得盯」时**，评估是否 insert / update，再跑相关命令。
- 读上下文时用 `memo view` 拉 top N，输出量恒定可控，不过度读取。

## 滚动日志

`index.json` 里的 `log` 数组是 Appender-only 的完整审计历史，每次 insert / update / drop 都会追加一条：
- `insert`：记下重要性、来源 agent、分配的序号。
- `update`：记下旧/新重要性（即注意力的提升或下降）与重排后的序号。
- `drop`：记下被挤掉/淘汰的那条（这是「注意力正在舍弃什么」的关键信号）。

用 `memo log --all` 一览全部历史，确认某 Agent 或某段时间，排查健忘/钻牛角尖的轨迹。

## 与其他文档的关系

- 这不替代 AGENTS.md（那是给 Agent 的长期行为准则）。
- 它是工作区「当下注意力」的滚动视图，服务于持续交付与自我纠偏。