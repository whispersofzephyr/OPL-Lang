# OPL-Lang

*A lean DSL for describing, generating, and evaluating equity-option strategies.*

![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)

📘 **[Live white-paper site →](https://whispersofzephyr.github.io/opl-lang/)**

---

### 📑 Specifications

| Language | File |
|----------|------|
| 🇺🇸 English | [English Spec (v1.0.0-rc6)](docs/opl-lang-spec-en-v1.0.0-rc6.md) |
| 🇨🇳 中文 | [中文规范 (v1.0.0-rc6)](docs/opl-lang-spec-zh-v1.0.0-rc6.md) |


📄 **Change Log**: [English](CHANGELOG.md) • [中文](CHANGELOG.zh.md)

---

## 🧠 What is OPL-Lang?

**OPL-Lang is a domain-specific language (DSL) that expresses option-strategy structure, intent, and constraints in a way that is both human-writable and machine-parsable.**

*It is not* a pricing engine or back-test framework.  
Think of it as **“SQL for option strategies”**: you describe *what* you want; execution layers decide *how* to price, back-test, or trade it.

Key design goals:

- ✅ **Readable** by traders & quants  
- ✅ **Deterministic** grammar for compilers  
- ✅ **Implementation-agnostic** (Python, Rust, C++, …)

---

## 📦 Repository Structure

```
docs/
  ├─ index.md                         ← GitHub Pages homepage
  ├─ opl-lang-spec-en-v1.0.0-rc6.md
  ├─ opl-lang-spec-en-v1.0.0-rc3.md  ← archived
  ├─ opl-lang-spec-zh-v1.0.0-rc6.md
  └─ opl-lang-spec-zh-v1.0.0-rc3.md
schema/
  └─ opl-lang-1.0.schema.json
samples/                              ← (optional) strategy examples
README.md
CHANGELOG.md
CHANGELOG.zh.md
LICENSE                                ← MIT license declaration
```

---

## 🏷️ Versioning

Release tags follow [Semantic Versioning 2.0.0](https://semver.org/):

- **`v1.0.0-rc6`** — current public candidate  
- **`v1.0.0-rc3`** — archived previous spec  
- Future: tag **`v1.0.0`** after schema & sample libraries stabilise

---

## ⚖️ License

MIT © 2025 OPL-Lang Authors

---

## 🤝 Contributors Welcome!

Interested in parsers, playgrounds, or runtime engines?  
Open an issue or submit a PR — prototypes & discussions are encouraged.

欢迎熟悉编程语言、编译器或量化系统的开发者参与贡献！


