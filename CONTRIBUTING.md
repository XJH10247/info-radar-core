# 贡献指南

感谢你对 **info-radar-core** 的关注。本项目是一个可安装的 Agent Skill：仓库根是 GitHub 协作文档，真正会被加载的技能包在 `skills/info-radar-core/`。

## 开发前须知

- **Skill 包内不要放 `README.md`**：说明写在 `SKILL.md` 与 `references/`，仓库级文档只放在仓库根。
- **目录名与 frontmatter `name` 必须一致**，且为 kebab-case（当前为 `info-radar-core`）。
- **渐进披露**：入口只保留流程与契约；细则、边界案例、长表放进 `references/`。
- **配置与模板可被用户覆盖**：`config/`、`templates/` 的改动要考虑“换人/换场景只改配置”的复用目标。

## 本地校验

若你安装了 skill-creator 类工具，可用其 `validate_skill.py` 校验本仓库 Skill 包：

```bash
python /path/to/skill-creator/scripts/validate_skill.py skills/info-radar-core
```

期望输出：`PASS` 且 0 error。没有该脚本时，请至少人工核对：

- [ ] `SKILL.md` 文件名大小写正确，以 `---` frontmatter 开头
- [ ] `name`、`description` 齐全；`description` 同时写清「做什么」和「何时使用」（含用户会说的触发语）
- [ ] frontmatter 无 XML 尖括号；`name` 不含保留前缀
- [ ] 正文引用的 `references/`、`config/`、`templates/` 文件都存在
- [ ] Skill 文件夹内无 `README.md`

## 提交类型

| 类型 | 说明 |
|---|---|
| Bug 修复 | 附脱敏后的原始通知文本、实际输出、期望输出 |
| 流程改进 | 说明动机；若改了五步流程，同步改 `references/examples.md` |
| 新场景配置 | 提交成套的 `sources.yaml` + `profile.md` 样例（信息源用 `example.edu.cn` 等占位） |
| 文档 | README / CONTRIBUTING / references 措辞与结构修正 |

## PR 规范

1. **一个 PR 只做一件事**，避免功能与大规模重排混在一起。
2. **不要提交运行时数据**：`state/`、`inbox/` 以及真实学校通知原文、个人画像中的隐私信息。
3. **改打分或分区规则时**，在 `references/examples.md` 补一条可复现的示例（输入 → 结构化 → 分数 → 分区）。
4. **保持示例可脱敏**：URL 使用 `example.edu.cn`，姓名学号等用占位符。
5. PR 描述写清：改动动机、影响的步骤/配置、验证方式。

## 行为准则

保持友善与专业；Issue 中聚焦可复现的问题与具体建议。骚扰、歧视性言论或恶意提交将被拒绝。

## 许可证

贡献内容默认与本仓库相同，采用 [MIT License](LICENSE)。
