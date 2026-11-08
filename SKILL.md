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
- 滚动历史落盘在同目录的 `memo/history/`（`{编号}-{操作}-{描述}.md`），不占 index.json。

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

# 总结清理：采集聚类 + 归并/重排/清理候选，供 agent 判断后落地
scripts/memo tidy
# 白日梦：从历史随手捞被遗忘/旧线索，当灵感与备忘补给
scripts/memo daydream [--n 3] [--mix]
# 业界对照：提炼当前关注主题，交有联网能力的 agent 去查当前最佳实践
scripts/memo watch [--topic <X>] [--top 3]
```

> 工作区默认取当前目录下的 `memo/`；可用环境变量 `MEMO_DIR` 指向任意工作区。
> Agent 来源 `--agent` 取自己的标识（如环境变量里的 agent id）。

## Playbook：场景化种子备忘

`playbooks/` 目录为常见任务场景预置了一套「种子备忘」（完成标准、纪律、核查清单）。选定一个场景时，把它的种子批量灌入当前工作区 memo，作为该任务的注意力基线。播放本是**一次性注入**，之后照常走 insert/update/drop。

```
scripts/memo playbook list                      # 列出已装场景
scripts/memo playbook init --name <slug> [--dry-run]   # 预览/灌入某场景（重复调用自动去重）
scripts/memo playbook route --query "<意图>"     # 未匹配时展示候选 + 提 issue 入口
```

已预置场景：

| slug | 场景 | 一句话 |
|---|---|---|
| `issue-to-ship` | Issue → 开发·测试·交付 | 一个 issue 交付成可验证改动 |
| `need-to-design` | 需求 → 可靠设计 | 零散需求收敛成可靠设计决策 |

**铺新场景**：在 `playbooks/<slug>.json` 里按两个例子写 `seeds`（每个含 desc/importance/note），脚本即可 `playbook init --name <slug>`。

**找不到匹配场景**：不要硬塞。引导用户到本仓库提 GitHub issue（建议标题 `new playbook: <意图>`），详见 `/memo` 斜杠指令的「路由纪律」。

## /memo 斜杠指令（自动路由）

pi 里通过 `~/.pi/agent/prompts/memo.md` 提供 `/memo` 命令：输入后 LLM 读当前对话上下午，把任务**语义路由到最匹配的 playbook** 并初始化。不发散时它连参数都不吃：`/memo` 或 `/memo source: issue #42`。

路由纪律：只路由一次；不匹配时**不硬塞**，提示提 issue；无法决定时占位初始化并在回复里注明。重试纪律：可逆操作大胆做；**同一操作重试 3 次仍失败即停止并通知用户**。

> /memo 的具体实现是一个 pi prompt template，代码在 `prompts/memo.md`（仓库内一份，安装时可拷贝到 `~/.pi/agent/prompts/`）。

## 梳理类命令：tidy / daydream / watch

三兄弟都是「广播」而非「自动魔法」：**脚本负责机械采集汇聚，agent(LLM)基于脚本给出的结构化素材做语义判断，再调 insert/update/drop 落地。** 保持「脚本存取、agent 判断」哲学。

- **`scripts/memo tidy`** —— 总结清理。脚本扫描 index+history，输出：①按来源/agent 聚类 ②低重要性清理候选 ③疑似重复/可归并(粗糙归一化匹配) ④最近 drop 的旧目标。**脚本不自己删**，只给候选；agent 决定每条 update 权重 / 归并 / drop。归并 = 挑一个保留、其余 drop、note 合并进保留项；重排由重要性自动驱动，无需额外命令。是滚动本本的「被挤掉的东西里，偶有值得重拾的」的补给。
- **`scripts/memo daydream [--n N] [--mix]`** —— 白日梦。从 history/ 随机捞被 drop 的旧线索(默认纯 drop 池，`--mix` 混入 insert/update)，并列出现在仍盯的 top3 对照。脚本只随机捞；agent 觉得值得重拾就 insert 回当前本子，纯当灵感也可以只记一笔 `insert`。
- **`scripts/memo watch [--topic X] [--top N]`** —— 业界对照。脚本提炼当前关注主题(默认取重要性最高几条的 desc/note，`--topic` 可显式覆盖)，**交给有联网能力的 agent** 据此查当前业界最佳实践。可复现做法/来源链接 `insert` 一条「外部最佳实践」，或与现有做法 `update` 对照。

> `watch` 不内置爬虫（保持纯 stdlib 无第三方依赖）：联网查询由 agent 自己的 Web 工具完成，脚本只给 query 原料。

## 开场协议（每次会话必做，零成本注意力引导）

**每次会话开始，先跑两下（脚本自动、输出恒定一小屏，不用我每次现场编排）：**

```
scripts/memo view --scope agent --top 10    # 1. 我现在的注意力焦点：个人 top 10
scripts/memo log --agent <my-id> --top 5     # 2. 最近的注意力变化：最近 5 次 insert/update/drop
```

这两条输出就是「现在该盯着什么 + 最近注意力怎么变的」。它们进上下文后就构成了我本次会话的注意力基线——
- 若发现该盯的没盯（健忘），或某条权重一枝独秀压倒主线（钻牛角尖），先自我纠正再动手。
- 用户随时扫 `memo/`（序号排序）或 `memo/history/`（时间排序）来诊断/干预我。

成本极低：`view` 恒一屏，`log` 只取最近几条，不读全量历史。

## 更新会话

- **每回合结束或每发现「这值得盯」时**，评估是否 insert / update，再跑相关命令。
- 写回时同一事实只改不堆（一致性规则），冲突必消解。

## 滚动日志

每次 insert / update / drop 都往 `memo/history/` 落一个文件，文件名 `{编号}-{操作}-{描述}.md`（方案A）：
- **编号**单调递增、绝不回写，文件夹顺序即时间先后。
- **操作**：`insert` / `update` / `drop`。
- **描述**：这条记录的 5~10 字实义描述（与笔记本身同一要求）。
- **时间戳**写在正文，不进标题。

`update` 记录旧/新重要性（注意力升降），`drop` 记下被挤掉/淘汰的那条（「注意力在舍弃什么」的关键信号）。

用 `memo log --all` 从文件夹按编号顺序一览全部历史；扫 `history/` 文件夹本身也能一眼看到滚动轨迹，排查健忘/钻牛角尖。

## 与其他文档的关系

- 这不替代 AGENTS.md（那是给 Agent 的长期行为准则）。
- 它是工作区「当下注意力」的滚动视图，服务于持续交付与自我纠偏。