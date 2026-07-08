# Skills 集

**一份 skill、两端通用**——本仓库里的每个 skill 都同时支持 **Claude Code** 和 **OpenAI Codex**。两者的 skill 格式基本一致（`skills/<name>/SKILL.md` 是主入口），只是加载路径不同：

| 工具 | 加载路径 |
|---|---|
| Claude Code | `~/.claude/skills/<name>/` 或 项目 `.claude/skills/<name>/` |
| Codex | `~/.codex/skills/<name>/` |

装到对应目录后，两个工具都会自动识别、自动触发。**同一份 skill 内容不用改。**

## 当前 skill

- **[`baisha-standards`](skills/baisha-standards/)** · 光山白鲨内部系统 AI 编码规范。架构选型（Lite / Standard / Scale）+ 业务代码规范（字段兜底 / 状态机 / 事务 / 审计 / 错误分类）+ 复用优先。适用于 MES、订单、生产、车间、采购、库存等业务系统。
- **[`feishu-web-article-copy`](skills/feishu-web-article-copy/)** · 把网页文章整理成本地 Markdown，并生成适合复制到飞书或 Lark Docs 的 HTML 页面。图片默认保留原始远程链接，不下载到本地。

## 目录结构

```text
skills/
├── baisha-standards/
│   ├── SKILL.md              主入口（Claude Code 和 Codex 都读这里）
│   ├── agents/
│   │   └── openai.yaml       Codex 特有：允许 $baisha-standards 隐式调用；Claude Code 会自动忽略
│   ├── references/           读的：判断依据类文档
│   │   ├── reuse-first.md    复用优先（7 步阶梯）
│   │   ├── stack-selection.md 选型十问 + 决策矩阵
│   │   ├── profile-lite.md   Vue + Fastify + SQLite 规格
│   │   ├── profile-standard.md Vue + FastAPI + MySQL 规格
│   │   ├── business-fields.md 建表规范
│   │   ├── state-machine.md  单据 6 态状态机
│   │   ├── transactions.md   事务 + 幂等 + 并发
│   │   ├── audit-trail.md    审计追溯
│   │   ├── errors.md         业务错 vs 系统错
│   │   ├── layout.md         目录规范
│   │   └── pitfalls.md       踩坑清单
│   └── templates/            抄的：代码骨架
│       ├── server-templates.md Fastify 后端骨架
│       ├── web-templates.md    Vue 前端骨架
│       ├── deploy-templates.md CentOS 7 部署
│       └── claude-md-template.md 项目根 CLAUDE.md 模板
└── feishu-web-article-copy/
    ├── SKILL.md
    ├── agents/
    │   └── openai.yaml
    └── scripts/
        └── md_to_feishu_html.py
```

## 安装

无论装到 Claude Code 还是 Codex，**skill 文件本身完全一样**，只需选一条路径复制过去。

### 装到 Claude Code

**用户级**（所有项目都能用）：
```bash
git clone https://github.com/haiyangyereps/skills.git /tmp/hy-skills
mkdir -p ~/.claude/skills
cp -R /tmp/hy-skills/skills/<skill-name> ~/.claude/skills/
```

**项目级**（只在特定项目里生效）：
```bash
mkdir -p 你的项目/.claude/skills
cp -R /tmp/hy-skills/skills/<skill-name> 你的项目/.claude/skills/
```

### 装到 Codex

```bash
git clone https://github.com/haiyangyereps/skills.git /tmp/hy-skills
mkdir -p ~/.codex/skills
cp -R /tmp/hy-skills/skills/<skill-name> ~/.codex/skills/
```

### 同时装到两个工具

```bash
git clone https://github.com/haiyangyereps/skills.git /tmp/hy-skills
mkdir -p ~/.claude/skills ~/.codex/skills
for s in baisha-standards feishu-web-article-copy; do
  cp -R /tmp/hy-skills/skills/$s ~/.claude/skills/
  cp -R /tmp/hy-skills/skills/$s ~/.codex/skills/
done
```

装完后：
- **Claude Code**：写业务代码时会自动加载 `SKILL.md`，无需显式调用
- **Codex**：可用 `$<skill-name>` 显式调用，或让 Codex 自动识别相关任务（`agents/openai.yaml` 里已配 `allow_implicit_invocation: true`）

## baisha-standards 快速上手

**适用场景**：光山白鲨信息中心的所有业务系统开发——MES、订单、生产、车间、采购、库存、质量、报表 等。

**核心内容**：

- **架构选型**：新项目开工先按 [stack-selection.md](skills/baisha-standards/references/stack-selection.md) 十问速判选 Profile（Lite / Standard / Scale），写 ADR
- **复用优先**：每次动手前按 [reuse-first.md](skills/baisha-standards/references/reuse-first.md) 7 步阶梯自查（骨架 / Element Plus / 已装依赖 / 标准库能做的就不写）
- **业务规范**（跨 Profile 一致）：字段兜底 5 类、单据状态机、事务边界、审计追溯、业务错 vs 系统错
- **完工自查**：SKILL.md §5 要求写完代码后必须结构化输出 8 项自查

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

**新增 skill**：在 `skills/` 下建一个新目录，至少包含：

1. `SKILL.md`（必需）—— frontmatter 里写 `name` 和 `description`，正文写触发条件和内容。**这是 Claude Code 和 Codex 都读的主入口**。
2. `agents/openai.yaml`（可选，只影响 Codex）—— 让 Codex 支持 `$<name>` 显式调用和隐式触发；Claude Code 会自动忽略此文件。
3. `references/`（推荐）—— 判断依据、参考资料
4. `templates/`（推荐）—— 可复用的代码骨架、模板

写完后在本 README 顶部的"当前 skill"清单里加一行说明。

**兼容性原则**：skill 内容里避免出现"只有 Claude Code 才能……"或"只有 Codex 才能……"这类绑死单一工具的措辞；在 markdown 里都用 "AI" / "coding agent" 这样的中性词，让同一份文件在两个工具里都能顺畅工作。
