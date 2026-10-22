---
name: eat
description: >
  知识内化引擎——像无脸男一样吸收外部知识并转化为自身能力。
  当用户提供 GitHub 仓库链接、文章 URL、代码片段、技术文档、Skill 目录，
  并希望将其"消化"为新 Skill、规则更新或已有 Skill 扩展时触发。
  典型触发："/eat URL"、"/eat 这段代码"、"帮我消化这个仓库"、
  "把这篇文章吃掉"、"吸收这个工作流"。
---
# Eat — 知识内化引擎
像《千与千寻》里的无脸男：吸收遇到的一切，把吃下去的东西变成自己的能力。
但不是来者不拒——消化的目标是留下精华，不是堆积数量。
## 工作流
### Phase 1: 输入识别与获取
判断输入类型，用对应方式获取完整内容：
| 输入类型 | 获取方式 |
|----------|----------|
| URL（文章/博客） | `web_fetch` 获取页面内容 |
| GitHub 仓库链接 | `web_fetch` 获取 README，重点分析仓库结构与核心逻辑 |
| GitHub PR/Issue | `web_fetch` 获取详情与讨论 |
| 代码片段 | 直接从用户消息中提取 |
| 本地文件路径 | `read_file` 读取内容 |
| Skill 目录路径 | `list_files` + `read_file` 扫描结构和内容 |
获取完成后，向用户确认："已获取内容，开始分析。"
### Phase 2: 深度分析
按 [references/analysis_framework.md](references/analysis_framework.md) 中的多维分析框架进行分析：
- 核心价值提取
- 适用场景映射
- 与现有能力的关系
### Phase 3: 影响扫描
扫描 `~/.<占位-codex,claude code>/` 下的 skills、rules、commands 目录：
```bash
# 扫描已有 Skills
ls ~/.<占位-codex,claude code>/skills/
# 逐个读取 SKILL.md 的 frontmatter 了解已有能力
```
列出与新内容有交集的已有组件，评估影响范围。
### Phase 4: 消化建议
基于分析结果，输出结构化建议：
```
## 🍽️ 消化分析报告
### 输入摘要
[一句话概括吃了什么]
### 核心价值
[提取的关键知识点/模式/工作流]
### 影响范围
[列出受影响的已有 Skill/规则]
### 建议消化形式
- [ ] 🆕 新建 Skill: [名称] — [理由]
- [ ] 📝 更新规则: [具体规则] — [修改内容]
- [ ] 🔄 扩展已有 Skill: [Skill 名] — [扩展内容]
- [ ] ❌ 不建议消化 — [理由]
### 具体方案
[详细描述将要做的修改]
```
**等待用户确认后才进入 Phase 5。**
### Phase 5: 执行消化
根据用户确认的方案执行：
- **新建 Skill**：调用 `<占位-codex,claude code>-skill-creator` skill 的流程创建
- **更新规则**：编辑对应的 rules 文件
- **扩展 Skill**：编辑已有 Skill 的 SKILL.md 或 references
执行完成后输出变更摘要。
## 过滤规则（When NOT to eat）
在 Phase 2 完成后立即检查，命中则跳过消化并告知用户原因：
1. **一次性查询** — "某 API 的参数是什么"之类，直接回答
2. **单行规则** — 一条 rule 能搞定的，建议直接加 rule，不建 Skill
3. **已知通用知识** — <占位-codex,claude code> 训练数据已覆盖的内容
4. **高度重叠** — 已有 Skill 覆盖 ≥80%，建议扩展而非新建
5. **易过时内容** — 特定版本的 changelog 等时效性信息
过滤时保持友好语气，解释为什么不消化，并给出替代建议。
## 资源
- **[references/analysis_framework.md](references/analysis_framework.md)** — 完整的多维分析框架、消化形式评估矩阵、影响扫描清单
