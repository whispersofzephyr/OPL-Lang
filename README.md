# OPL‑Lang

*A lean DSL for describing, generating, and evaluating equity option strategies.*

![](https://img.shields.io/badge/License-MIT-green.svg)

📘 [**View the live whitepaper site ➜**](https://whispersofzephyr.github.io/opl-lang/)

---

### 📑 Specifications

- 🇺🇸 [English Spec (v1.0.0)](https://chatgpt.com/c/docs/opl-lang-spec-en-v1.0.0.md)
- 🇨🇳 [中文规范 (v1.0.0)](https://chatgpt.com/c/docs/opl-lang-spec-zh-v1.0.0.md)

### 📐 JSON Schema

- [`schema/opl-lang-1.0.schema.json`](https://chatgpt.com/c/schema/opl-lang-1.0.schema.json)

---

## 🧠 What is OPL‑Lang?

**OPL‑Lang is a domain-specific language (DSL) designed to express options strategies — clearly, semantically, and machine-readably.**

It is not a pricing engine or a backtest framework.

Its goal is to describe the structure, intent, and risk constraints of a strategy in a way that is:

- ✅ Human-writable
- ✅ Machine-parsable
- ✅ Implementation-agnostic

**You talk to the machine in OPL‑Lang.** The machine then executes — using Python, Rust, C++, or whatever environment is appropriate.

> Think of OPL‑Lang as the "SQL of options strategies": express the logic, and let execution layers handle the rest.
> 

---

## 📦 Repository Structure

```
docs/
  ├─ index.md                   ← GitHub Pages homepage
  ├─ opl-lang-spec-en-v1.0.0.md
  └─ opl-lang-spec-zh-v1.0.0.md
schema/
  └─ opl-lang-1.0.schema.json
samples/                        ← (optional) strategy examples
README.md
LICENSE                         ← MIT license declaration

```

---

## 🏷️ Versioning

Release tags follow [Semantic Versioning 2.0.0](https://semver.org/):

- `v1.0.0` — current release
- Future versions may include sample libraries, runtime engines, or IDE integration

---

## ⚖️ License

MIT © 2025 OPL-Lang Authors

---

## 🤝 Collaborators Welcome

We're looking for contributors interested in implementing the OPL‑Lang parser, playground, or runtime.

If you're a developer passionate about DSLs, compilers, or financial systems, feel free to open an issue or get in touch via GitHub.

PRs, prototypes, and discussions are welcome.

欢迎熟悉编程语言、编译器、或者量化系统的开发者一起合作。有兴趣请提 Issue 或直接 PR。
