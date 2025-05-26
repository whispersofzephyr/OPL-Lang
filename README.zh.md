# OPL‑Lang 中文简介

![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)

📘 **[白皮书中英文网页版 →](https://whispersofzephyr.github.io/OPL-Lang/)**

🔗 - EN [English README →](README.md)

---

## 📘 项目概览

**OPL‑Lang 是一个用于构建和表达期权多腿策略结构的轻量领域专用语言（DSL）**。

它不是定价工具，也不是回测引擎，  
而是一套像 SQL 一样描述 *“你想要什么策略结构”* 的语言，  
再由后端系统决定 *“如何执行、定价或回测”*。

你可以将它理解为：  
📊 **策略结构的表达标准** + 🧠 **交易意图与机器逻辑之间的桥梁**

---

## 📑 文档入口

所有中英文内容已整理为网页系统，建议使用浏览器访问：

👉 [OPL-Lang GitHub Pages 中文/英文文档站点](https://whispersofzephyr.github.io/OPL-Lang/)

如果你更习惯直接查看 Markdown 文件，也可以从下方进入：

| 内容 | 中文 | English |
|------|------|---------|
| 核心规范 | [中文规范](docs/opl-lang-spec-zh-v1.0.0-rc6.md) | [English Spec](docs/opl-lang-spec-en-v1.0.0-rc6.md) |
| 语言结构 | [C‑O‑S 模型](docs/language-structure-zh.md) | [C-O-S Model](docs/language-structure-en.md) |
| 策略路径 | [路径 A/B/C](docs/application-paths-zh.md) | [Path A/B/C](docs/application-paths-en.md) |
| 更新日志 | [中文日志](docs/changelog.zh.md) | [Change Log](docs/changelog.en.md) |

---

## 🧠 语言核心思想

- 每个期权不是一个静态图形，而是由希腊值组成的动态响应系统；
- 多腿策略是一组这样的响应模块的结构组合；
- 意图、约束、结构三者应该解耦并标准化表达。

OPL‑Lang 用三层结构建模所有策略：

- `C`: 基元组件（如单腿期权 / 股票 / 现金）
- `O`: 操作单元（买/卖 指令）
- `S`: 策略结构（多腿组合 + 约束）

详情见：[语言结构](docs/language-structure-zh.md)

---

## 🏗️ 应用路径

OPL‑Lang 提供三种典型的策略生成方式：

- **路径 A**：由用户交互式逻辑树逐步构建策略（适合 App）
- **路径 B**：机器编译器根据约束全局优化策略结构
- **路径 C**：以目标行为（如 Delta 中性）为起点，反推策略结构

详情见：[策略路径文档](docs/application-paths-zh.md)

---

## 🔧 后续计划

- 开发路径 A 的策略构建 App
- 提供代码示例与接口文档
- 实现路径 B / C 的编译模块原型

---

## 🤝 欢迎合作

如果你对 DSL、编译器、量化策略引擎感兴趣，欢迎交流合作。

可以直接通过 Issue 或 PR 参与贡献，也欢迎微信/邮件联系作者。
