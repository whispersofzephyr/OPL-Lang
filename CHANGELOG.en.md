# 📌 Change Log – OPL‑Lang v1.0.0‑rc6

**Release Date**: 2025‑05‑24  
**Compared to**: [v1.0.0‑rc3](docs/opl-lang-spec-en-v1.0.0-rc3.md)

---

## 🔧 Structural & Semantic Refinements

- Clarified DSL scope as covering hybrid **stock + option** strategies.
- Strengthened wording: “not a pricing engine” made explicit.
- Adjusted formatting and consistency across all section headers and syntax blocks.

## ➕ New Features & Input Extensions

- Added `price_target` and `vol_target` as optional exit/funding inputs.
- Introduced support for **relative strike and tenor** (e.g. `+5%`, `‑30d`).
- Glossary updated with `K`, `T`, `L`, `Q` control parameters.

## ⚙️ Engine Behavior & Constraint Clarification

- `Stock` / `ETF` / `Future` defined to contribute **only** Δ = ±1; other Greeks are zero.
- Engines **MUST** handle contract multiplier logic explicitly (`1 option = 100 shares`).
- Clarified when `shift_strike` and `roll_tenor` are **NOP** for linear legs.
- `MapIntentToO` may include linear instruments when vega exposure is not needed.

## 🧠 Control & Risk Logic Updates

- Added `control_tune()` post-solve step to Path C (previously only implied).
- Refined canonicalisation of `Breakout` and `RangeBound` intents.
- Reinforced `max_risk` precedence over `premium_offset`; warning `W403` mentioned as degraded offset behavior.

## 🧪 Golden Samples Expanded

- 2 new strategy examples:
  - `scale_qty()` doubling a position.
  - Classic **covered call** example.

## 📐 Grammar & Validation

- Added grammar rules for `RelNUMBER`, `RelTIME` to support relative strike/tenor.
- New validation codes:
  - `E304` (lot-size mismatch)
  - `E305` (zero/negative quantity)
  - `W403` (premium offset degraded)

---

**See full release**: [opl-lang-spec-en-v1.0.0-rc6.md](docs/opl-lang-spec-en-v1.0.0-rc6.md)  
**Previous version**: [v1.0.0-rc3](docs/opl-lang-spec-en-v1.0.0-rc3.md)
