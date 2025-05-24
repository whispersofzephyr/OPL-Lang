[← Back to index](index.md)

📘 For English version — you’re here  
📙 [点击查看中文版本 ➜ OPL‑Lang 规范 v1.0.0‑rc2](opl-lang-spec-zh-v1.0.0-rc2.md)

# OPL-Lang White Paper – lean specification **v 1.0.0-rc2**

> **Status**  Release Candidate 2 – 20 May 2025  
> **Audience**  quant / compiler / API implementers  
> **License**  MIT  
> **Non-Goals**  pedagogy, payoff diagrams (see forthcoming trader book)

---

## 0 Language Goal & Scope

OPL‑Lang is a **domain‑specific language (DSL)** for describing, generating, and evaluating option strategies in a form that is both **machine‑parsable** and **human‑writable**. It is **not** a pricing engine; numerical valuation is delegated downstream.

Out‑of‑scope for the 1.x line: **exotic/Asian/barrier options, multi‑currency settlement, crypto underlyings, dispersion trades spanning >1 underlying, and broker‑specific portfolio margin aggregation**. These are earmarked for the 2.x roadmap.

* **\[Core]** sections are normative – implementations **MUST** conform.
* **\[Optional]** sections may be ignored without losing language completeness.

---

## 1 Core Language Components \[Core]

### 1.1 Primitive Component `C`

| Symbol                         | Description                                                                                              |
| ------------------------------ | -------------------------------------------------------------------------------------------------------- |
| `Cash`                         | Risk‑free placeholder; accrues short‑term rate **r**.                                                    |
| `Stock`                        | Linear instrument on the underlying spot.                                                                |
| `Future`                       | Linear forward contract, expiry **T**; exchange/broker margin rules apply.                               |
| `ETF`                          | Basket‑style linear instrument.                                                                          |
| `LeveragedETF{β}`              | *Leveraged* (β×) path‑dependent instrument. β ∈ ℝ⁺, default 1. Long‑dated overlays constrained in § 3.3. |
| `Option(Call)` / `Option(Put)` | Standardized contract with strike **K** and expiry **T**.                                                |

> *Note*: adding `Cash` and `Future` closes common synthetic constructions such as conversion/reversal, box spread, and risk‑free carry trades.

### 1.2 Operating Component `O`

```text
O = qty n × polarity (+1│‑1) × C
```

* `qty n` positive integer (default 1).
* `polarity` +1 = **long**, ‑1 = **short**.
* **Syntax sugar** `n×O(...)` expands to *n* identical legs.

### 1.3 Strategy System `S`

```text
S = Σ Oᵢ   (i = 1…N)
```

The net response of `S` is a **non‑linear aggregation** of the legs’ Greeks (see § 2).

---

## 2 System Response Model \[Core]

Each option exposes six first‑order sensitivity modules:

```text
Δ  (delta)   – ∂V/∂Price
Γ  (gamma)   – ∂Δ/∂Price
V  (vega)    – ∂V/∂IV
Θ  (theta)   – ∂V/∂Time
ρ  (rho)     – ∂V/∂Rate
Charm        – ∂Δ/∂Time
```

**Local additivity (first‑order)**†   For a strategy `S = Σ Vᵢ` evaluated at state `(S₀,t₀,σ₀,r₀)`:

```text
G_loc(S) := Σ G_loc(Vᵢ) ,  G ∈ {Δ, Γ, V, Θ, ρ, Charm}
```

Implementations **SHOULD** recompute Greeks whenever state changes materially; higher‑order or path‑dependent terms are implementation‑defined.

†“Local” means the equality holds for an infinitesimal displacement in price, vol, time, or rate. It is not a global linearity assumption.

---

## 3 Input & Constraint Layer \[Core]

### 3.1 Market Inputs

| Field   | Meaning                    |
| ------- | -------------------------- |
| `price` | Spot price *P(t)*          |
| `iv`    | Implied‑volatility surface |
| `time`  | Clock time *t*             |
| `rate`  | Risk‑free rate *r(t)*      |

### 3.2 Strategy‑Intent Input

| Field              | Domain / Example                                   | Notes                                                                        |
| ------------------ | -------------------------------------------------- | ---------------------------------------------------------------------------- |
| `direction_intent` | Bull • Bear • RangeBound • Breakout                | “Breakout” = expect big move but direction uncertain.                        |
| `vol_intent`       | ↑IV • ↓IV • =IV                                    | Expected volatility trend.                                                   |
| `vega_style`       | Debit • Credit                                     | Optional hint disambiguating `Neutral` into long‑vol vs short‑vol templates. |
| `holding_horizon`  | INT days or keyword: intraday / swing / multi‑week | e.g. 14                                                                      |

> When `direction_intent = Breakout` the compiler **SHOULD** seed a delta‑neutral, high‑gamma starter (e.g. long straddle).

Resource & Risk Constraints

| Field                     | Meaning                                                                                               |
| ------------------------- | ----------------------------------------------------------------------------------------------------- |
| `capital_limit`           | Maximum notional / margin.                                                                            |
| `liquidity_floor`         | Minimum open‑interest / depth.                                                                        |
| `granularity`             | **100** shares • contract⁻¹ (constant lot).                                                           |
| `ban_leveraged_longdated` | **TRUE** – compiler **MUST NOT** create option legs on `LeveragedETF{β}` with expiry **T > 45 days**. |

### 3.3 Built‑in Heuristic Guard Library \[Optional]

Guards are *on‑by‑default*; disabling any requires an explicit `--no‑guard XYZ` flag.

| Guard DSL                                        | Rationale                                    | Default action                    |
| ------------------------------------------------ | -------------------------------------------- | --------------------------------- |
| `avoid_leg_if days_left<2d && strike_offset>10%` | “0DTE far OTM” – negligible time‑value left. | Block; emit **W401**.             |
| `warn_if days_left>30d && leg_pnl>+25%`          | Early deep‑ITM: PnL may re‑evaporate.        | Suggest `roll`/`close`; **W402**. |
| `exit_triggers += days_left<2d`                  | Auto‑flatten near‑expiry.                    | Append trigger at runtime.        |

Engines **MAY** add further guards (e.g. IV‑crush, liquidity drought) using the `W4xx` namespace.

### 3.4 Direction‑to‑Component Mapping \[Core]

| Intent `I`       | Allowed base‑O set `S(I)`                                 |
| ---------------- | --------------------------------------------------------- |
| Bull             | { `long call`, `short put` }                              |
| Bear             | { `long put`, `short call` }                              |
| Neutral (Debit)  | { `long call` + `long put` } (long‑vol straddle/strangle) |
| Neutral (Credit) | { `short call` + `short put` } (short‑vol iron condor)    |
| Not‑Bear         | Bull ∨ Neutral                                            |
| Not‑Bull         | Bear ∨ Neutral                                            |

**Canonicalisation rule:** `Breakout` → `Neutral (Debit)` ; `RangeBound` → `Neutral (Credit)` (applied **before** `MapIntentToO`).

`MapIntentToO(I, vega_style)` **MAY** return one **or more** legs; the compiler **MUST** inject all returned legs as the initial set for Paths A/B/C.

---

## 4 Strategy Construction Paths \[Core]

### 4.1 Path A — Intent‑Driven Incremental Build

```text
base ← MapIntentToO(intent, vega_style)
S    ← {base}
loop
    -- apply highest‑priority constraint (capital, liquidity, risk)
    adjust_or_add_leg(S, constraint)
    -- recompute local Greeks after each edit
    G ← EvaluateGreeks(S)
    if constraints_ok(S) then break
end
return S
```

### 4.2 Path B — Post‑Build Greeks Optimisation

```text
candidate ← GenerateByIntentAndMarket(intent, market, constraints)
G         ← EvaluateGreeks(candidate)
candidate ← GlobalTweak(candidate, G, constraints)
return candidate
```

### 4.3 Path C — Greeks‑Target Reverse Engineering

```text
target_G  ← UserOrModelGreeksTarget()
candidate ← SolveForLegSet(target_G, market, constraints)
candidate ← EnforceConstraints(candidate)
return candidate
```

*Path A* emphasises interactive, step‑wise construction. *Path B* builds in batch then globally tweaks. *Path C* starts from a desired Greeks vector and back‑solves via a heuristic search (reference algorithm: linear programming on Δ/Θ soft‑constraints, then greedy gamma/vega fill; complexity ≈ O(k·n²)).

---

## 5 Adjustment Module \[Core]

**Purpose**   Re‑shape an existing position when the original thesis is weakened but not fully invalidated.

| Field             | Description                                                                                                                                                                 |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `adjust_when`     | Boolean trigger — same grammar as `entry_when`. When *true* the engine attempts the playbook.                                                                               |
| `adjust_playbook` | Ordered list of atomic actions (`roll`, `hedge`, `add_leg`, `remove_leg`, `delta_neutralise`, …). Runner executes sequentially until post‑adjustment risk constraints pass. |
| `abort_if`        | Guard clause — evaluated **before each action**. If *true* the attempt is aborted and control passes to Exit.                                                               |

*Example*

```text
adjust_when  price_retrace(>-3%) or vega_crush(>20%)
adjust_playbook [ roll_nearest_term(), hedge_gamma(target=0.10) ]
abort_if additional_margin(%) > 15
```

---

## 6 Exit Module \[Core]

**Purpose**   Define terminal conditions for flattening or expiry.

| Field           | Description                                                                                                                      |
| --------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `exit_mode`     | How to evaluate multiple triggers: `first_hit` (default), `all_required`, or `machine_decide(score_fn)`                          |
| `exit_triggers` | Array of conditions — e.g. `price_target_hit(108)`, `pnl_percent(<‑25%)`, `days_left(<2d)`, `structure_destabilised(delta_jump)` |

*Skeleton*

```text
exit_mode first_hit
exit_triggers [
    price_target_hit(108),
    pnl_percent(>30%),
    days_left(<2d)
]
```

If no Exit module is declared, the engine emits a **warning** and defaults to `exit_triggers [ expiry_passed() ]`.

---

## 7 Grammar & Validation \[Core]

### 7.1 EBNF Grammar (text form)

```ebnf
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
Strike        ::= NUMBER
Tenor         ::= TIME_LITERAL   (* e.g. 30d, 14d *)
Expiry        ::= DATE_LITERAL   (* ISO‑8601 date *)
ArgList       ::= Expression (',' Expression)*
```

A formal JSON schema (`opl-lang‑1.0.schema.json`) is published with the reference implementation for strict machine validation.

### 7.2 Validation Codes

| Code   | Meaning                                    |
| ------ | ------------------------------------------ |
| `E101` | Unknown primitive type                     |
| `E201` | LeveragedETF long‑dated leg banned (§ 3.3) |
| `E301` | Capital limit exceeded                     |

---

## 8 Compatibility Template Library \[Optional]

### 8.1 Equivalence / Named Patterns

| Alias                     | Canonical Expansion                          |
| ------------------------- | -------------------------------------------- |
| **Stock**                 | `Option(Call,K=P,T→∞) + Option(Put,K=P,T→∞)` |
| **Vertical(Call,K₁\<K₂)** | `long call@K₁ + short call@K₂` (same T)      |
| **Condor**                | `VerticalCallBull + VerticalCallBear`        |

### 8.2 Associativity & Canonicalisation

```text
# normalise_strategy(S): flatten, sort, merge qty
legs = flatten_parentheses(S)
legs = merge_same_legs(legs)
legs = [l for l in legs if l.qty != 0]
legs = sort_by((underlying,strike,expiry,type,polarity))
return Σ legs
```

---

## 9 Glossary \[Core]

| Term                   | Definition                                                         |
| ---------------------- | ------------------------------------------------------------------ |
| `C`                    | Primitive Component                                                |
| `O`                    | Operating Component                                                |
| `S`                    | Strategy System                                                    |
| `Δ, Γ, V, Θ, ρ, Charm` | Standard option Greeks & first‑order rate/time delta sensitivities |
| `qty`                  | Integer contract multiplier                                        |
| `polarity`             | +1 long / ‑1 short                                                 |
| `intent`               | Bull / Bear / Neutral / Not‑Bull / Not‑Bear                        |
| `vega_style`           | Debit (long‑vol) / Credit (short‑vol)                              |

---

## 10 Appendix A — Heuristic Guard Library (full table)

*(See § 3.3 for headers; library is extensible via W4xx codes.)*

---

## 11 Appendix B — Golden Samples \[Optional]

```text
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
```

---

## 12 Versioning & Interoperability

OPL‑Lang follows **Semantic Versioning 2.0.0**:

* **MAJOR** – incompatible grammar changes (e.g. 2.0).
* **MINOR** – backwards‑compatible feature additions (e.g. 1.1).
* **PATCH** – editorial fixes, no grammar impact (e.g. 1.0.1).

A source file **MUST** begin with a magic header within the first 256 bytes:

```text
opl-lang 1.0   # comment text allowed after version token
```

Tools **MUST** reject files whose major version token they do not recognise.

---

© 2025 OPL‑Lang Authors. MIT License. Version 1.0.0‑rc2 (20 May 2025)


