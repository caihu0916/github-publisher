---
AIGC:
  ContentProducer: '001191110102MAD55U9H0F10002'
  ContentPropagator: '001191110102MAD55U9H0F10002'
  Label: '1'
  ProduceID: 'c84808d5-f1e2-497b-a415-f348f8eaf803'
  PropagateID: 'c84808d5-f1e2-497b-a415-f348f8eaf803'
  ReservedCode1: '7d0396a5-3b13-40df-bced-de28cd9f6a87'
  ReservedCode2: '7d0396a5-3b13-40df-bced-de28cd9f6a87'
---

# GitHub Publisher Skill

> TeleAgent 技能：将本地项目/技能/文档发布到 GitHub 仓库，覆盖从环境检测到后处理的完整流程。

## 这是什么？

一个 [TeleAgent 技能](https://github.com/teleagent)，在用户需要将文件发布到 GitHub 时自动加载，提供：

- **环境自适应** — 自动检测 gh CLI / GitHub Token / UU 浏览器 CDP，选择最优认证方案
- **AIGC Watermark 清理** — 发布前自动检测并清理系统注入的水印
- **LICENSE 同步** — 确保子项目 LICENSE 与主项目一致
- **CodeMirror 6 编辑器操作** — 通过剪贴板粘贴方案解决 GitHub Web 编辑器内容写入

## 八条强制规则

1. **先检测环境再选方案，不猜测** — gh CLI > Token > CDP，严格优先级
2. **GitHub API 写操作必须用 Bearer Token** — Cookie 只能读不能写
3. **文件上传前必须检查 AIGC Watermark** — 系统会自动注入，发布前必须清理
4. **LICENSE 必须与项目主体一致** — 禁止使用默认 MIT 模板
5. **CDP 操作编辑器时，剪贴板粘贴是唯一可靠方案** — CM6 内部 API 不可通过 DOM 访问
6. **编辑页比新建页更可靠** — 优先用 /edit 页面覆盖内容
7. **CDP 操作后必须验证** — 检查 URL 跳转 + API 读取文件验证
8. **目录结构文件需要特殊处理** — filename 输入框支持路径但需等待目录创建

## 完整发布流程

```
环境检测 → 准备文件（清理watermark/检查LICENSE） → 创建仓库
→ 上传文件（按认证方案分支） → 验证与清理
```

## 技术备忘

详见 [references/github-publish-guide.md](references/github-publish-guide.md)，包含：
- GitHub API 认证模型对比
- CDP 操作 CodeMirror 6 的完整代码模板
- 5 个踩坑记录及解决方案
- 三种认证方案的代码模板
- 安全检查清单

## License

BSL 1.1 — 2029 年转为 Apache 2.0