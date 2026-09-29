---
name: info-radar-core
description: 把杂乱的学业/校园通知（教务网页、群消息、截图OCR）提炼成统一结构化条目，基于用户画像匹配打分，按 deadline 倒计时自动升级优先级，去重后生成飞书《学业晨报》。Use when the user asks to 整理/汇总校园通知、做信息雷达/晨报/日报、监控竞赛报名/讲座/实习招聘，或要做 deadline 盘点与提醒；即使用户没说"雷达"，只要是"多条通知 → 优先级清单/每日推送"的场景就应触发。Do NOT use for pure web scraping without scoring, or for general note-taking.
license: MIT
compatibility: 处理中文校园通知文本；运行时写入 state/seen.json，可选投递 inbox/；无需外部 API
metadata:
  author: info-radar-core contributors
  version: 1.0.0
  category: productivity
  tags: [campus, notifications, daily-brief, deadline, scoring, feishu]
---

# 学业信息雷达 · 核心引擎（info-radar-core）

把"多源杂乱通知" → "结构化条目" → "带匹配分与优先级的晨报"。
本 Skill 是端到端工作流的处理核心：换人复用时只改 `config/` 两个文件，流程不变。

## 运行基准

- **今天**：以运行环境的真实当前日期为准（记为"今天"），所有相对日期与 `days_left` 都锚定它，不要凭记忆猜测日期
- **时区**：默认 Asia/Shanghai，`profile.md` 可覆盖

## 输入契约

运行前读取以下文件（任一缺失时降级继续，见"降级策略"，不要中断）：

- `config/sources.yaml`：信息源清单（网址、分类、条目识别提示）
- `config/profile.md`：用户画像（专业、技术栈、关注方向、黑名单、提醒节奏）
- `state/seen.json`：近 14 天已推送条目，用于跨天去重（首次运行没有则视为空，本期结束必须写回）

待处理的原始通知文本：采集节点抓取的网页正文，或 `inbox/` 里的群通知文字/截图 OCR 结果。

## 处理流程（五步，逐条执行）

复制此清单跟踪进度：

```
- [ ] 步骤1 结构化提炼：每条原始文本抽成统一 JSON
- [ ] 步骤2 日期解析：标准化 deadline / event_date，算 days_left
- [ ] 步骤3 匹配打分：对照 profile 算 0-100 分 + reason
- [ ] 步骤4 去重合并：多源同事件合并；对照 seen.json 判定 新增/复推/静默
- [ ] 步骤5 分区渲染：按分数与倒计时升级规则渲染晨报，写回 seen.json
```

### 步骤1 结构化提炼

把每条杂乱文本抽成统一结构（缺失字段填 null，不要臆造）：

```json
{
  "title": "简洁标题",
  "type": "教务通知 | 竞赛报名 | 讲座活动 | 实习招聘",
  "source": "发布方（从落款/站点名识别）",
  "url": "原文链接（若有）",
  "deadline": "报名/提交截止，YYYY-MM-DD[ HH:MM]",
  "deadline_confidence": "high | medium | low（日期模糊时标 low，默认 high）",
  "event_date": "活动举办日（讲座/比赛当天），YYYY-MM-DD",
  "summary": "一句话讲清要做什么",
  "action": "建议动作（如'10/11前上官网填报名表'）",
  "matched_tags": ["命中的画像关键词"]
}
```

提炼要点：
- 全角符号归一化（`20：00` → `20:00`，全角数字/标点转半角）
- 落款（"XX中心宣"）归入 `source`，不进正文
- 截图/OCR 文本先清洗噪声（页眉页脚、表情、乱码）再抽取
- 同一事件出现在多个源 → 合并为一条：`source` 全部列出（如"教务处 + 学院群"），`deadline` 取更早者，只计一次分

### 步骤2 日期解析

- `deadline` 与 `event_date` 都标准化为 `YYYY-MM-DD`，含时间则 `YYYY-MM-DD HH:MM`
- 相对日期（"下周五""本月底"）按"今天"换算成绝对日期
- **无年份日期**（"10月11日"）：先按当年解析；若结果早于今天超过 14 天，视为次年（12月提到的"3月5日"多半是明年的）
- 模糊日期（"10月中旬"）取区间保守值，标 `deadline_confidence: low`，晨报注明"日期待确认"
- `days_left = 目标日 - 今天`（按天计；deadline 带时间时，当天 24:00 前仍算 days_left=0）

### 步骤3 匹配打分（0-100，权重可被 profile 覆盖）

| 维度 | 默认权重 | 评分依据 |
|---|---|---|
| 相关度 | 40 | 命中 profile 的专业/技术栈/关注方向的多少与强度 |
| 紧迫度 | 35 | 由 `min(days_left(deadline), days_left(event_date))` 决定，越近越高（分档曲线见 references/examples.md） |
| 价值度 | 25 | 对学业/保研/竞赛/简历的实际帮助 |

- 若 `profile.md` 提供 `scoring_weights`，用它覆盖默认 40/35/25（三项之和须为 100）
- **黑名单**：命中 `profile.md` 黑名单关键词的条目，相关度直接归 0 并丢弃——不进晨报，也不写入 seen.json
- **必须输出 `reason`**：一句话说明"为什么推荐给你"，引用命中的具体画像项（这是本 Skill 的核心价值，不可省略）

### 步骤4 去重合并（对照 state/seen.json）

为每条目生成去重键 `key`：URL 类先标准化——去掉 `http(s)://` 前缀、`?`/`#` 之后的参数与锚点、结尾的 `.htm/.html`（如 `cs.example.edu.cn/info/1034/19118`）；无 URL 则用 `type + title 前 12 字`。

| 状态 | 判定 | 处理 |
|---|---|---|
| 🆕 新增 | key 不在 seen.json | 正常进分区，标 🆕 |
| 持续追踪 | key 已存在，且分区未升级、未进入里程碑 | 只更新 seen.json 的 days_bucket，晨报在"持续追踪"区列一行（标题+剩余天数），条目多时可整段省略 |
| 倒计时升级 | 分区升级（🟡→🔴）或进入里程碑（剩3天/剩1天/当天） | **必须再次完整提醒**，标注"倒计时更新" |
| 已过期 | deadline/event_date 已过 | 移出晨报，seen.json 里标 done |

> 分区未变、仅 days 档位变化（如 8-14→4-7）**不算升级**，仍按持续追踪处理，避免临近截止时每天轰炸；完整复推只发生在分区升级或三个里程碑（剩3天/剩1天/当天）。
> 所有非丢弃条目（含 📋 常规区）都写回 seen.json；黑名单丢弃的不写。

seen.json 格式（保留近 14 天，更早的删除）：

```json
{
  "items": [
    {
      "key": "cs.example.edu.cn/info/1034/19118",
      "title": "传智杯校内选拔",
      "zone": "🟡",
      "days_bucket": "15-30",
      "first_seen": "2026-09-25",
      "done": false
    }
  ]
}
```

### 步骤5 分区渲染与状态写回

分区规则：
- **🔴 紧急待办**：`days_left ≤ 3`（或命中 profile 自定义提醒节奏）→ 强制置顶，无视分数；`d ≤ 1` 的条目带 🚨 强提醒标记（推送节点可据此 @所有人）
- **🟡 为你推荐**：`score ≥ 60`（可用 profile 的 `push_min_score` 覆盖）且未进紧急区，按分数降序
- **📋 常规通知**：其余条目，仅列标题+来源

- **倒计时自动升级**（关键）：同一条目随日期推移紧迫度上升，到"提前3天/提前1天"自动从 🟡 升入 🔴；无 `deadline` 但有 `event_date` 的活动类，按 `event_date` 计算紧迫度，避免"当天即过期"漏提醒
- 把排序后的条目套入 `templates/daily-brief.md` 渲染晨报文案，交由办公节点（飞书 Webhook / 多维表格）推送；飞书自定义机器人可用交互式卡片（`msg_type: "interactive"`）
- **空报处理**：本期无任何条目时，只发一句话"📭 今日无新增通知"，不要发空模板
- **写回 `state/seen.json`**：本期所有条目的 key、标题、分区、days 档位、过期标记

## 降级策略（任一配置缺失都不要中断运行）

- `sources.yaml` 缺失：仍处理传入文本与 inbox，晨报注明"未读取信息源清单"
- `profile.md` 缺失：用默认权重 40/35/25、无标签匹配，`reason` 退化为"未配置画像，默认收录"，并在晨报顶部提醒用户补配置
- `state/seen.json` 缺失：视为首次运行，全部按 🆕 处理

## 复用与扩展

- 换场景（招聘雷达/租房雷达）：只需改 `sources.yaml` 的 `category` 与 `profile.md`，本流程不变
- 打分权重、推送阈值、紧急天数、时区都可在 `profile.md` 覆盖默认值

## 参考

- 打分细则、紧迫度分档曲线与边界案例见 [references/examples.md](references/examples.md)
