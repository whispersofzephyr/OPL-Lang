# OPL‑Lang White Paper – lean specification **v 1.0.0‑rc6**

> Status Release Candidate 6 — 24 May 2025
> **Audience** quant / compiler / API implementers
> **License** MIT
> **Non‑Goals** pedagogy, payoff diagrams (see forthcoming trader book)

[← Return to Index](index.md) | 📙 [点击查看中文版本 ➜ OPL‑Lang 规范 v1.0.0‑rc6](opl-lang-spec-zh-v1.0.0-rc6.md)

---

## 0 Language Goal & Scope

**OPL‑Lang** is a **domain‑specific language (DSL)** for *describing, generating, and evaluating* hybrid **stock + option** strategies in a form that is both **machine‑parsable** and **human‑writable**. It is **not** a pricing engine; numerical valuation is delegated downstream.

*Out‑of‑scope for the 1.x line:* exotic/Asian/barrier options, multi‑currency settlement, crypto underlyings, dispersion trades spanning > 1 underlying, and broker‑specific portfolio‑margin aggregation.

* **\[Core]** sections are normative – implementations **MUST** conform.
* **\[Optional]** sections may be ignored without losing language completeness.

---

## 1 Core Language Components \[Core]

### 1.1 Primitive Component `C`

| Symbol                         | Description                                                                                              |
| ------------------------------ | -------------------------------------------------------------------------------------------------------- |
| `Cash` *(Optional)*            | Risk‑free placeholder; accrues short‑term rate *r*. Engines MAY disallow **short Cash**.                 |
| `Stock`                        | Linear instrument on the underlying spot.                                                                |
| `Future`                       | Linear forward contract, expiry **T**; exchange/broker margin rules apply.                               |
| `ETF`                          | Basket‑style linear instrument.                                                                          |
| `LeveragedETF{β}` *(Optional)* | *Leveraged* (β×) path‑dependent instrument. β ∈ ℝ⁺, default 1. Long‑dated overlays constrained in § 3.3. |
| `Option(Call)` / `Option(Put)` | Standardised contract with strike **K** and expiry **T**.                                                |

> Rows marked **Optional** MAY be ignored by lightweight engines.

### 1.2 Operating Component `O`

```opl
O = qty n × polarity (+1│-1) × C
```

* **`qty n`** positive integer (default 1). Control Module MAY adjust `n` via `scale_qty`.
  **Contract multiplier:** *1 option contract = 100 underlying shares*. Engines **MUST** translate option notionals using this constant (or the local exchange spec).
* **`polarity`** +1 = long, ‑1 = short.
* **Syntax sugar** `n×O(...)` expands to *n* identical legs.

### 1.3 Strategy System `S`

```opl
S = Σ Oᵢ   (i = 1…N)
```

`S` responds as a **non‑linear aggregation** of leg Greeks.

---

## 2 System Response Model \[Core]

```
Δ (delta)   – ∂V/∂Price
Γ (gamma)   – ∂Δ/∂Price
V (vega)    – ∂V/∂IV
Θ (theta)   – ∂V/∂Time
ρ (rho)     – ∂V/∂Rate
Charm       – ∂Δ/∂Time
```

For a strategy `S = Σ Vᵢ` at state `(S₀,t₀,σ₀,r₀)`:

```opl
G_loc(S) := Σ G_loc(Vᵢ)     G ∈ {Δ, Γ, V, Θ, ρ, Charm}
```

* Implementations **MUST** provide accurate Δ, Γ, V, Θ.
* Implementations **SHOULD** provide ρ and Charm for expiries > 45 d or if rate‑sensitivity / delta‑hedge automation is in scope; otherwise MAY return `null`.
* Implementations **SHOULD** recompute Greeks whenever state changes materially.

> **Linear legs:** `Stock` / `ETF` / `Future` contribute Δ = ±1 (signed by polarity) and Γ = V = Θ = ρ = Charm = 0.
>
> **Control–Greeks linkage.** Adjusting {strike *K*, expiry *T*, leg‑set *L*, quantity *Q*} re‑weighs the composite Greeks; engines MAY solve Greek targets directly over these controls.

---

## 3 Input & Constraint Layer \[Core]

### 3.1 Market Inputs

| Field   | Meaning                    |
| ------- | -------------------------- |
| `price` | Spot price *P(t)*          |
| `iv`    | Implied‑volatility surface |
| `time`  | Clock time *t*             |
| `rate`  | Risk‑free rate *r(t)*      |

### 3.2 Intent, Risk & Funding

| Field                     | Domain / Example                    | Role                                                                   |
| ------------------------- | ----------------------------------- | ---------------------------------------------------------------------- |
| `direction_intent`        | Bull • Bear • RangeBound • Breakout | Primary directional thesis                                             |
| `vol_intent` *(optional)* | ↑IV • ↓IV • =IV                     | Explicit volatility view; if omitted, engine infers from `vega_style`. |
| `vega_style`              | Debit • Credit                      | Distinguishes Neutral ⇢ long‑vol vs short‑vol                          |
| `holding_horizon`         | 10 d / intraday / swing             | Exposure window                                                        |
| `price_target` *(opt)*    | 120 USD                             | Exit guidance                                                          |
| `vol_target` *(opt)*      | 35 IV                               | Desired vol level                                                      |
| `max_risk` *(opt)*        | 2 000 USD                           | Hard cap on worst‑case loss                                            |
| `premium_offset` *(opt)*  | 50 %                                | Desired debit offset via credit legs                                   |

### 3.3 Resource & Liquidity

| Field                     | Meaning                                              |
| ------------------------- | ---------------------------------------------------- |
| `capital_limit`           | Max notionals / margin allowed                       |
| `liquidity_floor`         | Min open‑interest / depth required                   |
| `granularity`             | 100 shares · contract⁻¹ lot size                     |
| `ban_leveraged_longdated` | TRUE ⇒ forbid `LeveragedETF{β}` legs with *T* > 45 d |

### 3.4 Built‑in Heuristic Guards \[Optional]

Guards are **on‑by‑default**; disabling any requires `--no‑guard XYZ`.

| Guard DSL                                        | Rationale                       | Default action                  |
| ------------------------------------------------ | ------------------------------- | ------------------------------- |
| `avoid_leg_if days_left<2d && strike_offset>10%` | “0DTE far OTM” – little value   | Block; emit **W401**            |
| `warn_if days_left>30d && leg_pnl>+25%`          | Early deep‑ITM may re‑evaporate | Suggest `roll` or `close`; W402 |
| `exit_triggers += days_left<2d`                  | Auto‑flatten near expiry        | Append trigger at runtime       |

Engines **MAY** add further guards (e.g. IV‑crush, liquidity drought) under `W4xx`.

### 3.5 Direction‑to‑Component Mapping \[Core]

| Intent           | Allowed base‑O set `S(I)`                   |
| ---------------- | ------------------------------------------- |
| Bull             | { `long Stock`, `long call`, `short put` }  |
| Bear             | { `short Stock`, `long put`, `short call` } |
| Neutral (Debit)  | { `long call` + `long put` }                |
| Neutral (Credit) | { `short call` + `short put` }              |
| Not‑Bear         | Bull ∨ Neutral                              |
| Not‑Bull         | Bear ∨ Neutral                              |

**Canonicalisation rule:** `Breakout` → `Neutral (Debit)`; `RangeBound` → `Neutral (Credit)`.

`MapIntentToO` **MAY** return linear instruments (`Stock`/`Future`) when vega exposure is not required.

---

## 4 Strategy Construction Paths \[Core]

### 4.0 Control Module

| Symbol | Single‑leg        | Multi‑leg (N ≥ 1)      |
| ------ | ----------------- | ---------------------- |
| **K**  | strike price      | vector **K** = (K₁…Kₙ) |
| **T**  | expiry date       | vector **T** = (T₁…Tₙ) |
| **L**  | leg‑set (fixed 1) | {O₁…Oₙ}                |
| **Q**  | quantity          | vector **Q** = (q₁…qₙ) |

*For linear legs only ************`scale_qty`************, ************`add_leg`************, and ************`remove_leg`************ apply; ************`shift_strike`************ and ************`roll_tenor`************ are ************NOP************.*

Changing {K, T, L, Q} is semantically equivalent to inserting, removing, or rescaling legs and therefore re‑weights the composite Greeks.

### Atomic control actions

###

| Action Parameters Effect       |               |                          |
| ------------------------------ | ------------- | ------------------------ |
| `shift_strike(leg=i, ΔK)`      | absolute or % | Modify strike of leg *i* |
| `roll_tenor(leg=i, ΔT)`        | integer days  | Move expiry of leg *i*   |
| `replace_leg(i, newO)`         | —             | Swap leg *i* with `newO` |
| `add_leg(O)` / `remove_leg(i)` | —             | Expand / shrink **L**    |
| `scale_qty(leg=i, factor)`     | float         | Multiply quantity *qᵢ*   |

`shift_strike`, `roll_tenor`, `replace_leg`, `add_leg`, `remove_leg`, `scale_qty`. These verbs are available in every path loop and in the Adjustment Module.

### 4.1 Path A — Intent‑Driven Logic‑Tree

**Dependency chain**

```
market snapshot
   ↓
direction / (vol) intent
   ↓    choose initial leg(s)
market‑feedback  + control_tune
   ↓    fine‑tune Greeks
ensure_max_risk
   ↓
apply_premium_offset  (if requested)
   ↓
resource_liquidity_check
   ↓
loop until constraints_ok


```

(The **Resource/Liquidity** gate is last so earlier edits don’t produce false rejections.)

*Key points*

* **Risk‑first:** `max_risk` has priority; if unmet the compiler must add protective wings or reject.
* **Cost‑second:** `premium_offset` is best‑effort and can never violate `max_risk`.
* Control actions may occur at any loop iteration.
* The system Greeks are re‑evaluated after every structural edit.

### 4.2 Path B — Batch Build → Global Tweak

```
S₀ ← one_shot_generate(intent, constraints_with_basic_liquidity)  # light pre‑filter
S₁ ← global_tweak(S₀, target_Greeks, max_risk, premium_offset)
S₂ ← reality_check(S₁)    # full capital & liquidity gate
if !ok  then iterate or fallback(Path A)
return S₂


```

* Used when you prefer a single optimization pass rather than interactive edits.\*

### 4.3 Path C — Greeks‑Target Reverse Solve (market‑reaction first)

```
target_G ← user_or_model_vector()     # desired Δ,Γ,V,Θ,ρ …
candidate ← solve_leg_set(target_G, market, constraints)
── control_tune(candidate, policy)     # strike/tenor/leg/qty micro-adjust
candidate ← ensure_max_risk(candidate, max_risk)
candidate ← apply_premium_offset(candidate, premium_offset)

return reality_check(candidate)


```

After solving the leg set, engines **MAY** invoke `control_tune` for final micro-adjustment before risk and funding checks:

Path C is computationally heavy (LP/QP or heuristic search) and suited to ML / institutional engines.

---

## 5 Adjustment Module \[Core]

**Purpose** Re‑shape an existing position when the original thesis is weakened but not fully invalidated.

| Field Description |                                                                                                                                                                                                                        |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `adjust_when`     | Boolean trigger — same grammar as `entry_when`. When *true* the engine attempts the playbook.                                                                                                                          |
| `adjust_playbook` | Ordered list of atomic actions (`roll`, `hedge`, `add_leg`, `remove_leg`, `shift_strike`, `roll_tenor`, `scale_qty`, `delta_neutralise`, …). Runner executes sequentially until post‑adjustment risk constraints pass. |
| `abort_if`        | Guard clause — evaluated **before each action**. If *true* the attempt is aborted and control passes to Exit.                                                                                                          |

*Example*

```
adjust_when  price_retrace(>-3%) or vega_crush(>20%)
adjust_playbook [ roll_nearest_term(), hedge_gamma(target=0.10) ]
abort_if additional_margin(%) > 15


```

---

## 6 Exit Module \[Core]

**Purpose** Define terminal conditions for flattening or expiry.

| Field Description |                                                                                                                                  |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `exit_mode`       | How to evaluate multiple triggers: `first_hit` (default), `all_required`, or `machine_decide(score_fn)`                          |
| `exit_triggers`   | Array of conditions — e.g. `price_target_hit(108)`, `pnl_percent(<‑25%)`, `days_left(<2d)`, `structure_destabilised(delta_jump)` |

*Skeleton*

```
exit_mode first_hit
exit_triggers [
    price_target_hit(108),
    pnl_percent(>30%),
    days_left(<2d)
]


```

If no Exit module is declared, the engine emits a **warning** and defaults to `exit_triggers [ expiry_passed() ]`.

---

## 7 Grammar & Validation \[Core]

### 7.1 EBNF Grammar (text form)

```
Strategy      ::= Expression EOF
Expression    ::= Term (('+'|'-') Term)*
Term          ::= Multiplier '×'? Atom
Multiplier    ::= INTEGER
Atom          ::= Primitive | AliasCall | '(' Expression ')'
Primitive     ::= StockLeg | FutureLeg | CashLeg | ETFLeg | LeveragedETFLeg | OptionLeg
StockLeg      ::= ('long' | 'short')? 'Stock'
FutureLeg     ::= ('long' | 'short')? 'Future' '(' Expiry ')'
CashLeg       ::= ('long' | 'short')? 'Cash'
ETFLeg        ::= ('long' | 'short')? 'ETF'
LeveragedETFLeg::= ('long' | 'short')? 'LeveragedETF' '{' NUMBER '}'
OptionLeg     ::= ('long' | 'short')? 'Option' '(' CallPut ',' Strike ',' Tenor ')'
AliasCall     ::= IDENTIFIER '(' ArgList? ')'
CallPut       ::= 'Call' | 'Put'
Strike        ::= NUMBER | RelNUMBER  (* e.g. +5% *)
Tenor         ::= TIME_LITERAL | RelTIME  (* e.g. +30d *)
Expiry        ::= DATE_LITERAL
RelNUMBER     ::= ('+'|'-') NUMBER '%'
RelTIME       ::= ('+'|'-') NUMBER 'd'
ArgList       ::= Expression (',' Expression)*


```

A formal JSON schema (`opl-lang‑1.0.schema.json`) is published with the reference implementation for strict machine validation.

### 7.2 Validation Codes

| Code Meaning |                                                       |
| ------------ | ----------------------------------------------------- |
| `E101`       | Unknown primitive type                                |
| `E201`       | LeveragedETF long-dated leg banned (§ 3.3)            |
| `E301`       | Capital limit exceeded                                |
| `E302`       | max\_risk exceeded after final build                  |
| `E303`       | premium\_offset target unattainable                   |
| `E304`       | Quantity below lot-size granularity after `scale_qty` |
| `E305`       | Negative or zero quantity detected                    |

**Warnings**

| Code Meaning |                                              |
| ------------ | -------------------------------------------- |
| `W401`       | 0DTE far‑OTM leg blocked                     |
| `W402`       | Early deep‑ITM leg: suggest roll/close       |
| `W403`       | premium\_offset reduced to satisfy max\_risk |

---

## 8 Compatibility Template Library \[Optional]

### 8.1 Equivalence / Named Patterns

| Alias Canonical Expansion |                                              |
| ------------------------- | -------------------------------------------- |
| **Stock**                 | `Option(Call,K=P,T→∞) + Option(Put,K=P,T→∞)` |
| **Vertical(Call,K₁\<K₂)** | `long call@K₁ + short call@K₂` (same T)      |
| **Condor**                | `VerticalCallBull + VerticalCallBear`        |

### 8.2 Associativity & Canonicalisation

```
# normalise_strategy(S): flatten, sort, merge qty
legs = flatten_parentheses(S)
legs = merge_same_legs(legs)
legs = [l for l in legs if l.qty != 0]
legs = sort_by((underlying,strike,expiry,type,polarity))
return Σ legs


```

---

## 9 Glossary \[Core]

|   |
| - |

| TermDefinition         |                                                                     |
| ---------------------- | ------------------------------------------------------------------- |
| `C`                    | Primitive Component                                                 |
| `O`                    | Operating Component                                                 |
| `S`                    | Strategy System                                                     |
| `K`                    | Strike price (or vector)                                            |
| `T`                    | Expiry / tenor (or vector)                                          |
| `L`                    | Leg‑set                                                             |
| `Q`                    | Quantity (or vector)                                                |
| `Δ, Γ, V, Θ, ρ, Charm` | Standard option Greeks & first‑order rate/time delta sensitivities  |
| `qty`                  | Integer contract multiplier                                         |
| `polarity`             | +1 long / ‑1 short                                                  |
| `intent`               | Bull / Bear / Neutral / Not‑Bull / Not‑Bear                         |
| `vega_style`           | Debit (long‑vol) / Credit (short‑vol)                               |
| *Linear leg*           | Stock / ETF / Future; contributes Δ = ±1 (signed), other Greeks = 0 |

---

## 10 Appendix A — Heuristic Guard Library (full table)

*(See § 3.3 for headers; library is extensible via W4xx codes.)*

---

## 11 Appendix B — Golden Samples \[Optional]

```
# 1  Bull vertical (DSL)
S = long Option(Call,480,14d) + short Option(Call,500,14d)

# 2  Synthetic stock (JSON)
[
  {"qty":1,"side":"long","type":"call","strike":480,"exp":3650},
  {"qty":1,"side":"short","type":"put","strike":480,"exp":3650}
]

# 3  3× short puts via syntax sugar
S = 3×short Option(Put,450,30d)

# 4  Condor via alias
S = Condor(Call, K1=480, K2=520, T=30d)

# 5  Long call auto‑converted to debit vertical when premium_offset = 40%
intent           = Bull
premium_offset   = 40%
max_risk         = 1500 USD

# compiler output (illustrative):
S = long Option(Call,480,30d) + short Option(Call,520,30d)

# 6  Quantity scaling example
S = long Option(Call,480,30d)
adjust_playbook [ scale_qty(leg=1, factor=2.0) ]  # doubles notional

# 7  Covered call
S = long Stock + short Option(Call,520,30d)
```

---

## 12 Appendix C — Language Integration Flow (Natural Language ➜ Strategy ➜ Execution)

🧠 “The goal of OPL‑Lang is to become the interface language between human intent and automated strategy engines.”

```
Human natural‑language intent
          ↓
AI Semantic Parser (LLM / rules / few‑shot examples)
          ↓
OPL‑Lang Strategy Structure (intent + constraints + components)
          ↓
Execution Engine (Python / Rust / C++)
          ↓
Backtest / Live Order System / Portfolio Simulator


```

In this pipeline:

\- \*\*Humans\*\* express high-level intent (e.g. "I expect a breakout with rising volatility, low max loss").

\- \*\*AI\*\* parses the intent into OPL‑Lang, using predefined mappings or trained models.

\- \*\*OPL‑Lang\*\* provides the structural syntax, fully expressing direction, risk, horizon, capital, etc.

\- \*\*Execution engines\*\* consume this structure and transform it into live trades, backtests, or simulations.

## 13 Versioning & Interoperability

OPL‑Lang follows **Semantic Versioning 2.0.0**:

* **MAJOR** – incompatible grammar changes (e.g. 2.0).
* **MINOR** – backwards‑compatible feature additions (e.g. 1.1).
* **PATCH** – editorial fixes, no grammar impact (e.g. 1.0.1).

A source file **MUST** begin with a magic header within the first 256 bytes:

```
opl-lang 1.0   # comment text allowed after version token


```

Tools **MUST** reject files whose major version token they do not recognise.

---

© 2025 OPL‑Lang Authors. MIT License. Version 1.0.0‑rc6 (24 May 2025)

[← Return to Index](index.md)
