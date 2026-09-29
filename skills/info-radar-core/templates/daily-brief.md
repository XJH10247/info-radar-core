<!--
countdown_badge 取值：d=0「⏰今天 HH:MM 截止/今天举办」；d=1「🚨最后1天」；d≥2「还剩N天」
每个分区都必须有内容或空态回退，不允许输出空标题。
-->
# 🎓 学业晨报 · {{date}} {{weekday}}

> 今日处理 {{total}} 条：🆕 新增 {{new_count}} 条 ｜ 🔴 紧急 {{urgent_count}} 条 ｜ 生成于 {{generated_at}}

## 🔴 紧急待办

{{#each urgent}}
### {{countdown_badge}} {{title}}（{{#if is_upgrade}}倒计时更新{{else}}🆕{{/if}}）
- **做什么**：{{summary}}
- **谁发的**：{{source}}
- **截止/举办**：{{deadline_or_event_date}}（{{countdown_badge}}）
- **为什么找你**：{{reason}}
- **下一步**：{{action}}
- 原文：{{url}}

{{/each}}
{{#unless urgent}}✅ 暂无 3 天内的截止事项{{/unless}}

## 🟡 为你推荐（按分数降序）

{{#each recommended}}
**{{score}}分 · {{title}}** {{#if is_new}}🆕{{/if}}
{{deadline_or_event_date}}（{{countdown_badge}}）｜来源：{{source}}
{{reason}}
▶ {{action}} ｜ 原文：{{url}}

{{/each}}
{{#unless recommended}}（今日无达到推荐线的条目）{{/unless}}

## 📋 常规通知

{{#each routine}}- {{title}}（{{source}}）
{{/each}}
{{#unless routine}}（无）{{/unless}}

## 🔁 持续追踪（往期已推送，倒计时未变）

{{#each tracking}}- {{title}}：{{countdown_badge}}
{{/each}}
{{#unless tracking}}（无）{{/unless}}

---
由 info-radar-core 自动生成 · 画像与阈值见 config/profile.md
