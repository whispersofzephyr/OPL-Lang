[← Back to Index](index.md)

# 🧠 Core Philosophy & C‑O‑S Model

---

## 1. Foundational Philosophy

**OPL‑Lang** is based on a fundamental reinterpretation of what an option is.

> A single option is not a static payoff shape,  
> but a dynamic response system composed of Greek sensitivity modules.

Each option responds to market input — **price, volatility, time, rate** —  
through first-order partial derivatives known as Greeks: Δ, Γ, V, Θ, ρ, and others.  
These are not optional analytics — they are **the defining behaviors** of option instruments.

In this model:

- **Strike price** and **expiration** are not fixed parameters — they are **control interfaces**.
- Adjusting strike or tenor alters the activation profile of the instrument’s internal sensitivities.
- A multi-leg strategy becomes a **modular assembly** of response systems, tuned toward a specific market intent.

This perspective forms the semantic foundation for OPL‑Lang’s structural layers.

---

## 2. The C‑O‑S Structural Model

All strategy descriptions in OPL‑Lang are constructed from three conceptual layers:

| Layer | Symbol | Description |
|-------|--------|-------------|
| 🧩 **Primitive Component** | `C` | The base unit: a single instrument with defined sensitivity behavior. Typically an Option, Stock, or Cash leg. |
| 🔧 **Operational Unit** | `O` | A directed instruction to **buy or sell** a quantity of a `C` unit, at a specified strike and tenor. |
| 🏗 **Strategy Structure** | `S` | A set of `O` units combined with constraints, intent inputs, and optional tuning actions — representing the full logic of a position. |

This can be visualized as:

S = { O₁, O₂, ..., Oₙ }

Each Oᵢ = [ sign, qty, C ]
Where sign ∈ {+1, –1}, representing long or short direction

Each C = instrument + sensitivity model (Greeks)


The C‑O‑S structure allows OPL‑Lang to represent:

- ✅ Simple single-leg trades (as `S = {O}`),
- ✅ Complex multi-leg spreads,
- ✅ Dynamically tuned structures with fallback logic or exit control.

---

### 🔍 Terminology

| Term | Meaning |
|------|---------|
| `sign` | Direction of the trade: `+1` for **long (buy)**, `–1` for **short (sell)** |
| `qty` | Number of units (e.g., contracts) in this leg |
| `C` | A primitive component: an instrument with embedded sensitivity behavior |
| `O` | An operational instruction: what to do with a `C` |
| `S` | A strategy: multiple `O` instructions under constraints and intent |

---

## 3. Why This Structure Matters

Traditional strategy tools often treat options as static price components or payoff diagrams.  
**OPL‑Lang instead models strategies as responsive systems** — structures that react to market inputs and respect logical and risk constraints.

- **Humans** can use it via structured **Path A** to build template strategies like spreads, condors, or butterflies step by step, guided by market outlook and constraints.
- **Machines** can use refined **Path** A to build beyond templates, and use **Paths B and C** to synthesize or reverse-solve strategies toward a target Greek profile, payoff behavior, or capital objective.

In either case, OPL‑Lang is built on the belief that options are **behavioral components**,  
and that real-world strategy logic must be expressed not as static definitions, but as semantic structures driven by sensitivity and intent.

---

## 4. Generation Paths and Application Layer

While OPL‑Lang focuses on structural expression, its real-world power is best realized when paired with systems that can **generate**, **adjust**, and **exit** strategies based on user input or market conditions.

To support different usage scenarios, OPL‑Lang is designed to be compatible with multiple generation pathways:

- 🧩 **Path A** — a logic-tree strategy builder guided by human input (e.g., UI apps)
- 🔧 **Path B** — a machine compiler that assembles structures from intent and constraints
- 🔍 **Path C** — a reverse solver that constructs strategies from a target Greek profile or payoff behavior

These paths reflect increasing levels of automation and can be combined with lifecycle logic to create fully autonomous, constraint-respecting strategy agents.

For full details on implementation layers and usage, see  
📘 [Deployment Paths →](application-paths-en.md)

---
## 5. Lifecycle Control: Adjustment and Exit

While the core structure of OPL‑Lang focuses on initial strategy composition (`S = { O₁, O₂, ... }`), the language also provides built-in mechanisms for **managing the strategy across time** — enabling **reactivity and closure**.

Two key modules support this:

- 🔄 **Adjustment Module** — allows a strategy to adapt when initial assumptions weaken but are not invalidated. It includes:
  - `adjust_when`: boolean triggers for adjustment attempts
  - `adjust_playbook`: ordered list of actions (e.g., roll, hedge, rebalance)
  - `abort_if`: guards that cancel adjustment and escalate to exit

- ⏹ **Exit Module** — defines how and when a strategy is terminated. It includes:
  - `exit_mode`: evaluation logic (`first_hit`, `all_required`, or `machine_decide`)
  - `exit_triggers`: condition list (price, PnL, time, etc.)

These modules make OPL‑Lang not just a language for static structures,  
but a full **strategy lifecycle DSL** — capable of describing, adapting, and closing positions programmatically.

This empowers both human-guided and machine-driven agents to maintain strategies over time with clear, declarative logic.

OPL‑Lang is designed not just to describe what a strategy is,  
but also how it is **built**, **adjusted**, and **completed** — **from entry to exit**.

[← Back to Index](index.md)
