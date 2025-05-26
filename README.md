# OPL-Lang

*A lean DSL for describing, generating, and evaluating equity-option strategies.*

![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)

📘 **[Live white-paper site →](https://whispersofzephyr.github.io/OPL-Lang/)**

🔗 - 🇨🇳 [中文介绍 →](README.zh.md)

---

## 🧭 Overview

**OPL-Lang is a lightweight domain-specific language (DSL) designed to express the structure, intent, and constraints of multi-leg option strategies in a modular and machine-readable form.**

It enables strategies to be described like “SQL for options” — specifying *what* you want, while allowing pricing/backtesting/execution layers to determine *how* to fulfill it.

The language is intended to bridge the gap between trader intuition and automated strategy logic.

**OPL-Lang is not** a pricing engine or simulator. It is a structural foundation that supports both human-guided workflows (logic tree) and machine-driven compilation or reverse solving.

---

## 📑 Documentation

| Type | 🇺🇸 English | 🇨🇳 中文 |
|------|------------|---------|
| Core Spec | [Spec (v1.0.0-rc6)](docs/opl-lang-spec-en-v1.0.0-rc6.md) | [规范 (v1.0.0-rc6)](docs/opl-lang-spec-zh-v1.0.0-rc6.md) |
| Language Structure | [C-O-S Model](docs/language-structure-en.md) | [C‑O‑S 模型](docs/language-structure-zh.md) |
| Strategy Paths | [Path A/B/C](docs/application-paths-en.md) | [策略路径](docs/application-paths-zh.md) |
| Change Log | [English](docs/changelog.en.md) | [中文](docs/changelog.zh.md) |

More documents will be added under `/docs` as the project grows.

---

## 🧠 Language Philosophy

OPL-Lang is built on the belief that:

- **Options are dynamic response modules**, composed of Greeks;
- **Strategy structures are modular assemblies** of such primitives;
- **Intent, constraint, and structure** should be separated and formalized.

To reflect this, all strategies in OPL-Lang are defined using the **C‑O‑S model**:

- `C`: Primitive Components (option/stock/cash with sensitivity)
- `O`: Operational Units (buy/sell instructions)
- `S`: Strategy Structure (combined legs with constraints)

See [language structure →](docs/language-structure-en.md)

---

## 🏗️ Deployment Paths

OPL‑Lang supports 3 deployment paths:

- **Path A**: Logic-tree guided human UI for stepwise strategy building
- **Path B**: Machine compiler with global constraint + Greek optimization
- **Path C**: Reverse compiler based on target behavior (e.g. delta-neutral)

See [application paths →](docs/application-paths-en.md)

---

## 📦 Repository Layout

```
docs/
  ├─ index.md                   ← GitHub Pages homepage
  ├─ opl-lang-spec-*.md         ← Core specs (en/zh)
  ├─ language-structure-*.md    ← Structure / model explanation
  ├─ application-paths-*.md     ← Strategy path documentation
  ├─ changelog.*.md             ← Update history
schema/
  └─ opl-lang-1.0.schema.json
samples/                        ← Optional examples
README.md
LICENSE                         ← MIT license declaration
```

---

## 🏷️ Versioning

Follows [Semantic Versioning 2.0.0](https://semver.org/):

- **`v1.0.0-rc6`** — current candidate
- **`v1.0.0`** — target milestone (pending schema/sample stabilization)

---

## 🚧 Coming Soon

- Logic-tree App for strategy building (Path A)
- Code samples and integrations
- Compiler modules for Paths B/C

---

## 🤝 Contributors Welcome!

Interested in parsers, strategy engines, or UI apps?

Open an issue or submit a PR — collaboration is highly encouraged.

欢迎熟悉编程语言、编译器或量化系统的开发者参与贡献！


