---
name: "create-branch"
description: "创建功能分支的标准流程"
priority: 90
---
# 创建功能分支
## 分支命名规范
### 功能分支
- `feature/JIRA-123-user-login` - 新功能开发
- `feature/add-payment-gateway` - 功能描述
### 修复分支
- `bugfix/JIRA-456-fix-login-error` - Bug 修复
- `hotfix/critical-security-patch` - 紧急修复
### 其他分支
- `refactor/optimize-database-queries` - 重构
- `docs/update-api-documentation` - 文档更新
## 创建分支流程
1. **更新主分支**
  ```bash
  git checkout main
  git pull origin main
  ```
2. **创建新分支**
  ```bash
  git checkout -b feature/your-feature-name
  ```
3. **推送到远程**
  ```bash
  git push -u origin feature/your-feature-name
  ```
## 注意事项
- 分支名使用小写字母和连字符
- 包含 JIRA 票号（如果有）
- 描述要简洁明了
- 避免使用特殊字符
---
💡 **提示**: 使用 `{{SKILL:create-branch}}` 在模板中引用此规范。
