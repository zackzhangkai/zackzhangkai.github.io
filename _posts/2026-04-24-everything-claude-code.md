---
layout: post
title: Everything Claude Code 完全指南
tags: AI文档
---

> AI Agent Harness 性能优化系统 | 16.6万+ Stars

## 项目简介

**Everything Claude Code**（简称 ECC）是由 **Anthropic Hackathon 获奖者**打造的**性能优化系统**，被誉为"不只是配置文件"的完整解决方案。

| 统计 | 数量 |
|-------|------|
| Stars | 16.6万+ |
| Forks | 2.5万+ |
| Contributors | 170+ |
| 语言生态 | 12+ |

## 核心特性

### 🤖 48+ 代理 Agents
预置的生产级代理，覆盖编码、审查、调试、安全扫描等场景。

### 🛠️ 183+ 技能 Skills
即插即用的技能模块，支持深度学习、Web开发、文档生成等。

### 💾 记忆优化 Memory
Hooks 自动保存/加载跨会话上下文，解决长对话记忆衰减问题。

### 📚 持续学习 Continuous Learning
自动从会话中提取模式，转化为可复用的技能。

### 🛡️ 安全扫描 AgentShield
红队/蓝队/审计三轮对抗分析，保障 Agent 配置安全。

## 快速安装

### 方式一：Claude Code 插件市场（推荐）

```bash
# 添加市场
/plugin marketplace add https://github.com/affaan-m/everything-claude-code

# 安装插件
/plugin install everything-claude-code@everything-claude-code
```

### 方式二：手动安装

```bash
# 克隆仓库
git clone https://github.com/affaan-m/everything-claude-code.git
cd everything-claude-code

# 复制配置
cp -r .claude/* ~/.claude/
```

## 核心目录结构

| 目录 | 说明 |
|------|------|
| `.agents/` | 子代理定义 |
| `commands/` | 命令实现 |
| `hooks/` | 会话钩子 |
| `skills/` | 技能模块 |
| `rules/` | 编码规范 |

## 相关资源

- 🌍 官网: https://ecc.tools
- 🐙 GitHub: https://github.com/affaan-m/everything-claude-code
- 📦 npm: https://www.npmjs.com/package/ecc-universal
- 🖥️ 在线教程: https://claude-for-everything.vercel.app

---

> Everything Claude Code 不仅仅是一套配置文件，而是一个完整的 **AI Agent 性能优化生态系统**。