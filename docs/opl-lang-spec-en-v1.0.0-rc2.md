opl-lang 1.0   # magic header for files

**For Chinese version click here** / **中文版本请点击这里**  
ZH 📄 [OPL-Lang 规范 – 精简版 v1.0.0-rc2](opl-lang-spec-zh-v1.0.0-rc2.md)

# OPL-Lang White Paper – lean specification **v 1.0.0-rc2**

> **Status**  Release Candidate 2 – 20 May 2025  
> **Audience**  quant / compiler / API implementers  
> **License**  MIT  
> **Non-Goals**  pedagogy, payoff diagrams (see forthcoming trader book)

---

## 0 Language Goal & Scope

OPL‑Lang is a **domain‑specific language (DSL)** for describing, generating, and evaluating option strategies in a form that is both **machine‑parsable** and **human‑writable**.  It is **not** a pricing engine; numerical valuation is delegated downstream.

Out‑of‑scope (v 1.x): **exotic options, multi‑currency settlement, crypto underlyings, portfolio‑level margin netting**.  These are earmarked for a future 2.x line.

* **\[Core]** sections are normative – implementations **MUST** conform.
* **\[Optional]** sections may be ignored without losing language completeness.

---

## 1 Core Language Components \[Core]

### 1.1 Primitive Component `C`

| Symbol                         | Description                                                                                               |
| ------------------------------ | --------------------------------------------------------------------------------------------------------- |
| `Cash`                         | Risk‑free placeholder; accrues risk‑free rate `r`.                                                        |
| `Stock`                        | Linear instrument on the underlying spot.                                                                 |
| `Future`                       | Cash‑settled forward contract, expiry **T**; margin style is broker‑defined.                              |
| `ETF`                          | Basket‑style linear instrument.                                                                           |
| `LeveragedETF{β}`              | *Leveraged* (β×) path‑dependent instrument. β ∈ ℝ⁺, default 1.  Long‑dated overlays constrained in § 3.3. |
| `Option(Call)` / `Option(Put)` | Standardized contract with strike **K** and expiry **T**.                                                 |

> *Rationale*: adding `Cash` and `Future` closes common synthetic‑construction gaps (e.g. conversion/reversal, box spreads).

### 1.2 Operating Component `O`

```text
O = qty n × polarity (+1│‑1) × C
```

* `qty n` positive integer (default 1).
* `polarity` +1 = **long**, ‑1 = **short**.
* **Syntax sugar** `n×O(...)` expands to *n* identical legs.

### 1.3 Strategy System `S`

```text
S = Σ Oᵢ   (i = 1…N)
```

The net response of `S` is a **non‑linear aggregation** of the legs’ Greeks (see § 2).

---

## 2 System Response Model \[Core]

Each option embeds six local response modules

```text
Δ (delta)   – ∂V/∂Price
Γ (gamma)   – ∂Δ/∂Price
V (vega)    – ∂V/∂IV
Θ (theta)   – ∂V/∂Time
ρ (rho)     – ∂V/∂Rate
Charm       – ∂Δ/∂Time
```

**Local additivity (first‑order)**†   For a strategy `S = Σ Vᵢ` evaluated at state `(S₀,t₀,σ₀,r₀)`:

```text
G_loc(S) := Σ G_loc(Vᵢ)   G ∈ {Δ, Γ, V, Θ, ρ, Charm}
```

Implementations **SHOULD** recompute Greeks whenever state changes materially; higher‑order or path‑dependent terms are implementation‑defined.

† “Local” means the equality holds for an infinitesimal price/IV/time/rate displacement.  It is not a global linearity assumption.

---

## 3 Input & Constraint Layer \[Core]

### 3.1 Market Inputs

| Field   | Meaning                            |
| ------- | ---------------------------------- |
| `price` | Spot price *P(t)*                  |
| `iv`    | Implied volatility surface *IV(t)* |
| `time`  | Clock time *t*                     |
| `rate`  | Risk‑free rate *r(t)*              |

### 3.2 Strategy‑Intent Input

| Field              | Domain / Example                                   | Notes                                                                                                 |
| ------------------ | -------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| `direction_intent` | Bull • Bear • RangeBound • Breakout                | “Breakout” = expect big move but direction uncertain.                                                 |
| `vol_intent`       | ↑IV • ↓IV • =IV                                    | Expected volatility skew direction.                                                                   |
| `vega_style`       | Debit • Credit                                     | Optional hint that disambiguates `Neutral` direction between long‑vol (Debit) and short‑vol (Credit). |
| `holding_horizon`  | INT days or keyword: intraday / swing / multi‑week | 14                                                                                                    |

> When `direction_intent = Breakout`, compiler **SHOULD** seed a delta‑neutral, high‑gamma starter (e.g. long straddle).

Resource & Risk Constraints

| Field                     | Meaning                                                                                               |
| ------------------------- | ----------------------------------------------------------------------------------------------------- |
| `capital_limit`           | Maximum notional / margin.                                                                            |
| `liquidity_floor`         | Minimum open‑interest / depth.                                                                        |
| `granularity`             | **100** shares • contract⁻¹ (constant).                                                               |
| `ban_leveraged_longdated` | **TRUE** – compiler **MUST NOT** create option legs on `LeveragedETF{β}` with expiry **T > 45 days**. |

### 3.3 Built‑in Heuristic Guard Library \[Optional]

Moved from Core for readability.  Guards are *on‑by‑default*; disabling any requires an explicit `--no‑guard XYZ` CLI flag.

| Guard                                                | Rationale                                                    | Default Action                    |
| ---------------------------------------------------- | ------------------------------------------------------------ | --------------------------------- |
| `avoid_leg_if days_left < 2d && strike_offset > 10%` | “0DTE far OTM”: tiny ladle, almost no chance to scoop value. | Block; emit `W401`.               |
| `warn_if days_left > 30d && leg_pnl > +25%`          | Early deep‑ITM: unrealised PnL likely to re‑evaporate.       | Suggest `roll` / `close`; `W402`. |
| `exit_triggers += days_left < 2d`                    | Ensure near‑expiry positions are flattened.                  | Auto‑append trigger.              |

Implementations **MAY** enrich the library (IV‑crush, liquidity‑drought, etc.) using the `W4xx` namespace.

---

### 3.4 Direction‑to‑Component Mapping \[Core]

| Intent `I`       | Allowed base‑O set `S(I)`      |
| ---------------- | ------------------------------ |
| Bull             | { `long call`, `short put` }   |
| Bear             | { `long put`, `short call` }   |
| Neutral (Debit)  | { `long call` + `long put` }   |
| Neutral (Credit) | { `short call` + `short put` } |
| Not‑Bear         | Bull ∨ Neutral                 |
| Not‑Bull         | Bear ∨ Neutral                 |

`MapIntentToO(I, vega_style)` **MAY** return one **or more** legs; the compiler **MUST** inject all returned legs as the initial set when Paths A/B/C start.

---

## 4 Strategy Construction Paths \[Core]

(unchanged — see original spec for algorithms A/B/C).

---

## 5 Adjustment Module \[Core]

(Same as rc1, but the runner **MUST** evaluate `abort_if` **before** executing the full playbook and MAY short‑circuit on any fatal margin breach.)

---

## 6 Exit Module \[Core]

(Unchanged.)

---

## 7 Grammar & Validation \[Core]

### 7.1 EBNF Grammar (text form)

```ebnf
Strategy      ::= Expression EOF
Expression    ::= Term (('+'|'-') Term)*
Term          ::= Multiplier '×'? Atom
Multiplier    ::= INTEGER
Atom          ::= Primitive | AliasCall | '(' Expression ')'
Primitive     ::= StockLeg | OptionLeg | FutureLeg | CashLeg | ETFLeg | LeveragedETFLeg
StockLeg      ::= ('long' | 'short')? 'Stock'
FutureLeg     ::= ('long' | 'short')? 'Future' '(' Expiry ')'
CashLeg       ::= ('long' | 'short')? 'Cash'
OptionLeg     ::= ('long' | 'short')? 'Option' '(' CallPut ',' Strike ',' Tenor ')'
AliasCall     ::= IDENTIFIER '(' ArgList? ')'
CallPut       ::= 'Call' | 'Put'
Strike        ::= NUMBER
Tenor         ::= TIME_LITERAL   (* e.g. 30d, 14d *)
Expiry        ::= DATE_LITERAL   (* ISO‑8601 *)
ArgList       ::= Expression (',' Expression)*
```

A formal JSON schema (`opl‑lang‑1.0.schema.json`) is published in the reference repository for strict machine validation.

### 7.2 Validation Codes

| Code   | Meaning                                    |
| ------ | ------------------------------------------ |
| `E101` | Unknown primitive type                     |
| `E201` | LeveragedETF long‑dated leg banned (§ 3.3) |
| `E301` | Capital limit exceeded                     |

---

## 8 Compatibility Template Library \[Optional]

(unchanged apart from being relocated after grammar.)

---

## 9 Glossary \[Core]

(added `ρ`, `Charm`, `vega_style`).

---

## 10 Appendix A — Heuristic Guard Library (continued)

(contains the guard table originally in § 3.3.)

---

## 11 Appendix B — Golden Samples \[Optional]

(unchanged — sample listing from rc1.)

---

## 12 Versioning & Interoperability

OPL‑Lang follows **Semantic Versioning 2.0.0**:

* MAJOR = incompatible grammar changes (e.g. v2.0).
* MINOR = backwards‑compatible feature additions (e.g. v1.1).
* PATCH = editorial fixes, no grammar impact (e.g. v1.0.1).

A source file **MUST** begin with a magic header in the first 256 bytes:

```text
opl-lang 1.0   # comment text allowed after version token
```

Tools **MUST** reject files whose major version token they do not recognise.

---

© 2025 OPL‑Lang Authors. MIT License.  Version 1.0.0‑rc2  (20 May 2025)

---
