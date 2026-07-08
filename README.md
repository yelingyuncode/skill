# Skills 集

这个仓库用于管理和分享可复用的 AI 编码 skill（**Codex 和 Claude Code 都能用**——两者的 skill 格式一致：`skills/<name>/SKILL.md`）。

## 当前 skill

- **[`baisha-standards`](skills/baisha-standards/)** · 光山白鲨内部系统 AI 编码规范。架构选型（Lite / Standard / Scale）+ 业务代码规范（字段兜底 / 状态机 / 事务 / 审计 / 错误分类）+ 复用优先。适用于 MES、订单、生产、车间、采购、库存等业务系统。
- **[`feishu-web-article-copy`](skills/feishu-web-article-copy/)** · 把网页文章整理成本地 Markdown，并生成适合复制到飞书或 Lark Docs 的 HTML 页面。图片默认保留原始远程链接，不下载到本地。

## 目录结构

```text
skills/
├── baisha-standards/
│   ├── SKILL.md              (主入口)
│   ├── reuse-first.md
│   ├── stack-selection.md
│   ├── profile-lite.md
│   ├── profile-standard.md
│   ├── business-fields.md
│   ├── state-machine.md
│   ├── transactions.md
│   ├── audit-trail.md
│   ├── errors.md
│   ├── layout.md
│   ├── server-templates.md
│   ├── web-templates.md
│   ├── deploy-templates.md
│   ├── pitfalls.md
│   ├── claude-md-template.md
│   └── INSTALL.md            (给同事的安装说明)
└── feishu-web-article-copy/
    ├── SKILL.md
    ├── agents/
    │   └── openai.yaml
    └── scripts/
        └── md_to_feishu_html.py
```

## 安装

### Claude Code

**用户级**（所有项目都能用）：

```bash
mkdir -p ~/.claude/skills
cp -R skills/<skill-name> ~/.claude/skills/
```

**项目级**（只在特定项目里生效）：

```bash
mkdir -p 你的项目/.claude/skills
cp -R skills/<skill-name> 你的项目/.claude/skills/
```

### Codex

```bash
mkdir -p ~/.codex/skills
cp -R skills/<skill-name> ~/.codex/skills/
```

安装后在 Codex 里可用 `$<skill-name>` 调用；Claude Code 会自动识别。

## baisha-standards 快速上手

适合场景：**光山白鲨信息中心的所有业务系统开发**——MES、订单、生产、车间、采购、库存、质量、报表 等。

核心内容：

- **架构选型**：新项目开工先按 [stack-selection.md](skills/baisha-standards/stack-selection.md) 十问速判选 Profile（Lite / Standard / Scale），写 ADR
- **复用优先**：每次动手前按 [reuse-first.md](skills/baisha-standards/reuse-first.md) 7 步阶梯自查（骨架 / Element Plus / 已装依赖能做的就不写）
- **业务规范**（跨 Profile 一致）：字段兜底 5 项、单据状态机、事务边界、审计追溯、业务错 vs 系统错

装完在 Claude Code 里写 MES / 订单 / 生产 相关代码时自动触发。详细安装见 [INSTALL.md](skills/baisha-standards/INSTALL.md)。

## feishu-web-article-copy 快速上手

适用场景：

- 复制网页文章到飞书文档
- Markdown 图片链接粘贴到飞书后不渲染
- 希望保留图片原始链接而不是下载图片
- 需要生成浏览器可打开、可全选复制的 HTML

核心流程：

1. 提取网页正文内容和图片原始 URL
2. 生成本地 Markdown 文件，图片使用 `![](原始图片链接)`
3. 使用脚本把 Markdown 转成 HTML
4. 在浏览器打开 HTML，等待图片显示后，全选复制渲染后的页面到飞书

脚本示例：

```bash
cd skills/feishu-web-article-copy
python3 scripts/md_to_feishu_html.py article.md -o article-复制到飞书.html --title "文章标题"
```

可选参数：

- `--title`：设置 HTML 页面的标题
- `--keep-link-captions`：在图片下方显示原图链接，仅在需要把链接明文保留到文档里时使用

注意事项：

- 默认不下载图片，只保留原始远程图片链接
- 不建议把 HTML 源码直接复制到飞书；应该复制浏览器里渲染后的页面
- 如果浏览器里能显示图片，但粘贴到飞书后仍不显示，通常是飞书拒绝了对应远程图片源。此时只能改为上传图片，或使用飞书 API 处理图片

## 贡献

新增 skill：在 `skills/` 下建一个新目录，包含 `SKILL.md`（frontmatter 里写 `name` 和 `description`），然后在本 README 顶部的"当前 skill"清单里加一行。
