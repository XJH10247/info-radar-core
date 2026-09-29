# info-radar-core · 学业信息雷达核心引擎

把「多源杂乱通知」→「结构化条目」→「带匹配分与优先级的晨报」。

一个面向大学生的 **AI Agent Skill**：将教务网页、群消息、截图 OCR 等杂乱通知提炼为统一结构化条目，依据个人画像做 0–100 匹配打分，按 deadline 倒计时自动升级优先级，跨天去重后渲染成飞书《学业晨报》。换人复用或换场景（招聘雷达 / 租房雷达）只需修改 skill 包内 `config/` 两个文件，处理流程不变。

## 功能特性

- **结构化提炼**：全角符号归一化、落款识别归源、OCR 噪声清洗、多源同事件自动合并
- **日期解析兜底**：相对日期（“下周五”）、无年份日期、模糊日期（“10月中旬”）统一换算为绝对日期并标注置信度
- **画像匹配打分**：相关度 40 + 紧迫度 35 + 价值度 25，权重可被画像覆盖；黑名单关键词直接拦截；每条必附“为什么推荐给你”
- **倒计时自动升级**：剩 3 天从 🟡 推荐区升入 🔴 紧急区并完整复推，剩 1 天强提醒，避免“当天即过期”漏报
- **跨天去重**：`state/seen.json` 持久化已推送状态，同一事件不重复轰炸，只保留倒计时追踪行
- **降级容错**：任一配置文件缺失都降级继续运行，不中断晨报产出
- **模板化输出**：分区渲染 + 空态回退，兼容飞书自定义机器人交互式卡片

## 仓库结构

本仓库采用 **GitHub 文档与 Skill 包分离** 的布局：仓库根存放协作与发布文件，可安装的 Skill 位于 `skills/info-radar-core/`（目录名与 frontmatter `name` 一致）。

```text
info-radar-core/                      # GitHub 仓库根
├── README.md                         # 本文件：项目说明与使用指引
├── CONTRIBUTING.md                   # 贡献指南
├── LICENSE                           # MIT 许可证
├── .gitignore                        # 忽略运行时状态与本地私有数据
└── skills/
    └── info-radar-core/              # Skill 包（安装时复制这一层）
        ├── SKILL.md                  # 技能入口：输入契约、五步流程、输出契约
        ├── config/
        │   ├── sources.yaml          # 信息源清单（请替换为你的学校/场景）
        │   └── profile.md            # 用户画像：专业、技术栈、关注方向、黑名单、可选阈值
        ├── references/
        │   └── examples.md           # 打分细则、紧迫度分档、边界案例、去重判定示例
        └── templates/
            └── daily-brief.md        # 晨报渲染模板（分区 + 空态回退）

# 以下目录在 Skill 运行时生成，已 gitignore，不会入库：
#   skills/info-radar-core/state/     # seen.json 跨天去重状态
#   skills/info-radar-core/inbox/     # 可选：群通知文字 / 截图 OCR 投放入口
```

## 安装

前置：安装支持 Agent Skills 发现的客户端（如 MiMo Desktop、Claude Code 等，识别 `SKILL.md` 与 `.agents/skills/` 或等价技能目录）。

1. 克隆本仓库：

```bash
git clone https://github.com/<your-username>/info-radar-core.git
```

2. 将 **Skill 包** 复制到技能目录（不要把整个仓库根拷进去，仓库根含 README/LICENSE，不是 Skill 文件夹）：

```bash
# 用户级（所有项目可用）
cp -r info-radar-core/skills/info-radar-core ~/.agents/skills/info-radar-core

# 项目级（仅当前项目可用）
cp -r info-radar-core/skills/info-radar-core <your-project>/.agents/skills/info-radar-core
```

Windows（PowerShell）示例：

```powershell
git clone https://github.com/<your-username>/info-radar-core.git
Copy-Item -Recurse .\info-radar-core\skills\info-radar-core "$env:USERPROFILE\.agents\skills\info-radar-core"
```

3. 修改配置（两处，均为注释齐全的示例文件）：
   - `config/sources.yaml`：替换为你学校的教务处、学院官网、讲座页、就业网等真实信息源
   - `config/profile.md`：填写专业、技术栈、关注方向与黑名单；按需取消可选配置注释（权重 / 推送阈值 / 紧急天数 / 时区）

4. （可选）接入端到端流水线：上游采集节点把网页正文或 `inbox/` 内 OCR 文本交给本 Skill；下游办公节点用飞书自定义机器人 Webhook（支持 `msg_type: "interactive"` 卡片）定时推送晨报。

## 使用

在 Agent 会话中用自然语言触发：

```text
把这段群通知整理进今天的晨报：【计算机学院……报名截止时间为 10 月 11 日 20：00……】
帮我盯着教务处和学院官网，每天早上 8 点出一份学业晨报
盘点一下最近两周所有竞赛和实习的 deadline
```

Skill 会按五步流程处理并跟踪进度：

1. 结构化提炼  
2. 日期解析  
3. 匹配打分  
4. 去重合并  
5. 分区渲染  

晨报分为 🔴 紧急待办 / 🟡 为你推荐 / 📋 常规通知 / 🔁 持续追踪，并在 `state/seen.json` 持久化去重状态。无新增时只回一句「📭 今日无新增通知」。

打分细则、紧迫度分档曲线与边界案例见 [skills/info-radar-core/references/examples.md](skills/info-radar-core/references/examples.md)。

## 配置与扩展

| 需求 | 做法 |
|---|---|
| 调整打分权重 | `config/profile.md` 底部取消 `scoring_weights` 注释（三项之和须为 100） |
| 调整推送 / 紧急阈值 | 同上，`push_min_score`、`urgent_days` |
| 换场景（招聘雷达 / 租房雷达） | 只改 `sources.yaml` 的 `category` 与 `profile.md`，流程不变 |
| 自定义晨报版式 | 编辑 `templates/daily-brief.md` |
| 查看打分细则与边界案例 | [references/examples.md](skills/info-radar-core/references/examples.md) |

## 贡献

欢迎 Issue 与 PR，细则见 [CONTRIBUTING.md](CONTRIBUTING.md)。摘要：

- **Bug 报告**：请附脱敏后的原始通知文本、实际输出与期望输出
- **新场景配置**：考研升学、招聘求职、租房生活等场景的 `sources.yaml` + `profile.md` 样例优先收录
- **提交规范**：一个 PR 聚焦一件事；修改 SKILL.md 流程时请同步更新 `references/examples.md` 中的示例

## 许可证

[MIT](LICENSE) © 2026 info-radar-core contributors
