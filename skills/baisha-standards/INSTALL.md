# 安装说明（给白鲨内部同事）

这是光山白鲨信息中心的 **AI 编码规范 skill**，装到 Claude Code 里之后，你写业务代码（MES / 订单 / 生产 / 车间 / 采购 / 库存 / 质量 等）时会自动带上公司规范。

## 一分钟安装

### 方案 A · 只在当前项目里生效（推荐）

在你的项目根目录下：

```bash
cd 你的项目根目录
mkdir -p .claude/skills
# 把整个 baisha-standards 目录放到这里
# （比如从压缩包解压出来）
mv ~/Downloads/baisha-standards ./.claude/skills/
```

然后在这个项目里跑 Claude Code 就自动加载了。**别的项目不受影响**。

### 方案 B · 所有项目都生效

```bash
mkdir -p ~/.claude/skills
mv ~/Downloads/baisha-standards ~/.claude/skills/
```

装完重启 Claude Code。

## 验证

新开 Claude Code 会话，问一句"用 baisha-standards"，它会自动读 `SKILL.md`。或者直接跟 AI 说：

> 帮我在 MES 里加一个"报工"接口

如果 AI 主动提到"按 baisha-standards 规范"、"5 字段兜底"、"状态机"等词，就是装好了。

## 目录内容

```
baisha-standards/
├── SKILL.md                     ← 主入口，AI 会自动读
├── reuse-first.md               ← 复用优先（不重复造轮子）
├── stack-selection.md           ← 项目选型：Lite / Standard / Scale
├── profile-lite.md              ← Vue + Fastify + SQLite 详细规格
├── profile-standard.md          ← Vue + Python FastAPI + MySQL 详细规格
├── business-fields.md           ← 表字段兜底 5 项 + 单号规则
├── state-machine.md             ← 单据状态机 (6 态 + 8 条合法迁移)
├── transactions.md              ← 事务边界 + 幂等 + 并发
├── audit-trail.md               ← audit_log 表 + 必记场景
├── errors.md                    ← 业务错 vs 系统错
├── layout.md                    ← 目录结构 + 命名规则
├── server-templates.md          ← Lite 后端骨架
├── web-templates.md             ← Vue 前端骨架
├── deploy-templates.md          ← CentOS 7 部署脚本
├── pitfalls.md                  ← 踩坑清单
└── claude-md-template.md        ← 新项目根 CLAUDE.md 模板
```

## 使用建议

- **新项目开工**：先按 `stack-selection.md` 做选型 + 写 ADR
- **每次开发前**：让 AI 走一遍 `SKILL.md §2` 的判断阶梯
- **PR review 时**：用 `reuse-first.md §7` 的 6 条 checklist 自查
- **遇到问题**：先翻 `pitfalls.md` 踩坑清单

## 版本

当前 v2.0.0（2026-07-08 由信息中心叶海洋维护）。

有问题反馈给信息中心。
