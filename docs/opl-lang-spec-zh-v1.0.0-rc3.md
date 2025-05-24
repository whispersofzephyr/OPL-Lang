[← 回到目录](index.md)

📘 [Click here for English version ➜ OPL‑Lang Whitepaper v1.0.0‑rc3](opl-lang-spec-en-v1.0.0-rc3.md)

# OPL‑Lang 白皮书 —— 精简规范 **v 1.0.0-rc3**

> 状态：发布候选版 3 —— 2025 年 5 月 20 日
> 
> 
> **目标读者**：量化人员 / 编译器开发者 / API 实现者
> 
> **许可证**：MIT
> 
> **非目标**：教学用途、盈亏图表（详见即将发布的交易者手册）
> 

---

## 0 语言目标与适用范围

OPL‑Lang 是一个**领域专用语言（DSL）**，用于以机器可解析、人类可书写的形式描述、生成和评估期权策略。它**不是**定价引擎，数值定价任务被下游系统处理。

1.x 版本不包含的内容：**奇异/亚式/障碍期权、多币种结算、加密货币标的、跨多个标的的离差交易、经纪商特定的投资组合保证金聚合**。这些将在 2.x 路线图中考虑。

- 标注为 **[核心]** 的部分为规范性内容 —— 实现 **必须** 遵守。
- 标注为 **[可选]** 的部分可忽略，忽略不会影响语言的完整性。

---

## 1 语言核心组件 [核心]

### 1.1 原始组件 `C`

| 符号 | 描述 |
| --- | --- |
| `Cash` | 无风险占位符；按短期利率 **r** 计息。 |
| `Stock` | 标的现货的线性工具。 |
| `Future` | 远期线性合约，到期时间 **T**；需遵循交易所/券商保证金规则。 |
| `ETF` | 类似篮子的线性工具。 |
| `LeveragedETF{β}` | *杠杆型*（β×）路径相关工具。β ∈ ℝ⁺，默认值为 1。长周期覆盖在 §3.3 中有限制。 |
| `Option(Call)` / `Option(Put)` | 标准化期权合约，行权价 **K**，到期时间 **T**。 |

> 说明：Cash 和 Future 的引入可构建常见的合成策略，例如转换/反转、箱体价差、无风险套利等。
> 

### 1.2 操作组件 `O`

```
O = 数量 n × 极性 (+1│‑1) × C

```

- `数量 n`：正整数（默认值为 1）。
- `极性 polarity`：+1 表示**多头**，-1 表示**空头**。
- **语法糖**：`n×O(...)` 展开为 *n* 个相同的腿。

### 1.3 策略系统 `S`

```
S = Σ Oᵢ   (i = 1…N)

```

策略 `S` 的整体响应是各腿 Greeks 的**非线性聚合**（见 §2）。

---

## 2 系统响应模型 [核心]

每个期权具有六个一阶敏感性模块：

```
Δ（delta）   – ∂V/∂价格
Γ（gamma）   – ∂Δ/∂价格
V（vega）    – ∂V/∂隐含波动率
Θ（theta）   – ∂V/∂时间
ρ（rho）     – ∂V/∂利率
Charm        – ∂Δ/∂时间

```

**局部可加性（一阶）**†  对于某一状态 `(S₀, t₀, σ₀, r₀)` 下的策略 `S = Σ Vᵢ`：

```
G_loc(S) := Σ G_loc(Vᵢ) ,  G ∈ {Δ, Γ, V, Θ, ρ, Charm}

```

实现 **应当** 在状态显著变化时重新计算 Greeks。高阶或路径相关项的定义由具体实现决定。

†“局部”指该等式仅在价格、波动率、时间或利率发生无穷小变动时成立。并非全局线性假设。

---

## 3 输入与约束层 [核心]

### 3.1 市场输入项

| 字段 | 含义 |
| --- | --- |
| `price` | 现价 *P(t)* |
| `iv` | 隐含波动率曲面 |
| `time` | 当前时间 *t* |
| `rate` | 无风险利率 *r(t)* |

### 3.2 意图、风险与资金

> 编译器在读取 Greeks 前，首先会解析这些人类层面的目标。
> 

| 字段 | 范围 / 示例 | 作用 |
| --- | --- | --- |
| `direction_intent` | Bull • Bear • RangeBound • Breakout | 主要方向性判断 |
| `vol_intent` | ↑IV • ↓IV • =IV | 波动率预期 |
| `vega_style` | Debit • Credit | 区分中性策略是多波动率还是空波动率 |
| `holding_horizon` | *INT* 天 / 日内 / 波段 / 多周 | 持仓周期 |
| `max_risk` *(可选)* | 2000 USD / 3% | **最坏情况下的最大亏损上限**。默认 = ∞（无限制）。编译器 **必须** 添加保护腿或缩小价差以满足此限制。 |
| `premium_offset` *(可选)* | 50% | 期望通过卖出腿来抵消的权利金比例。默认 = 0%。编译器 **可** 添加空头腿，但**不得**违反 `max_risk` 限制。 |

当 `direction_intent = Breakout` 时，编译器 **应当** 初始化一个 delta 中性、高 gamma 的起始结构（如多头跨式）。

**冲突规则**：若无法在不违反 `max_risk` 的情况下满足 `premium_offset`，编译器 **必须** 优先满足 `max_risk`，并输出警告 **W403 premium_offset_degraded**。

**资源与流动性限制**

（每次编辑后检查）

| 字段 | 含义 |
| --- | --- |
| `capital_limit` | 允许的最大名义金额 / 保证金 |
| `liquidity_floor` | 最低持仓兴趣 / 深度要求 |
| `granularity` | 100 股 / 每份合约（单位批量） |
| `ban_leveraged_longdated` | TRUE ⇒ 禁止使用到期时间大于 45 天的 `LeveragedETF{β}` 腿 |

### 3.3 内置启发式防护库 [可选]

防护项**默认启用**；禁用时需显式加上 `--no-guard XYZ` 参数。

| 防护规则 DSL | 原因说明 | 默认动作 |
| --- | --- | --- |
| `avoid_leg_if days_left<2d && strike_offset>10%` | “0DTE 且远 OTM” —— 剩余时间值几乎为零。 | 阻止加入；发出 **W401** 警告。 |
| `warn_if days_left>30d && leg_pnl>+25%` | 提前进入深度 ITM：浮盈可能消失。 | 建议 `roll` 或 `close`；发出 **W402** 警告。 |
| `exit_triggers += days_left<2d` | 到期前自动平仓。 | 在运行时追加触发条件。 |

引擎 **可** 增加自定义防护规则（如波动率崩塌、流动性枯竭等），使用 `W4xx` 命名空间。

### 3.4 方向与组件映射 [核心]

| 意图 `I` | 允许的基础操作集合 `S(I)` |
| --- | --- |
| Bull | { `long call`, `short put` } |
| Bear | { `long put`, `short call` } |
| Neutral（借方） | { `long call` + `long put` }（多波动率跨式/宽跨式） |
| Neutral（贷方） | { `short call` + `short put` }（空波动率铁鹰式） |
| Not‑Bear | Bull ∨ Neutral |
| Not‑Bull | Bear ∨ Neutral |

**规范化规则**：`Breakout` → `Neutral（借方）`；`RangeBound` → `Neutral（贷方）`（此规则在 `MapIntentToO` 之前执行）。

`MapIntentToO(I, vega_style)` **可以** 返回一个或多个腿；编译器 **必须** 注入所有返回腿作为路径 A/B/C 的初始结构。

---

## 4 策略构建路径 [核心]

### 4.1 路径 A —— 基于意图的逐步逻辑树

```
base ← MapIntentToO(intent, vega_style)        # 主腿 / 目的腿
S    ← {base}
loop
    ── market_feedback(S, snapshot)            # 根据价格/IV 细调 Greeks
    ── ensure_max_risk(S, max_risk)            # 若超限 → 添保护腿/缩价差
    ── apply_premium_offset(S, premium_offset) # 若需融资 → 添收权利金副腿
    ── liquidity_capital_check(S)              # 资金 & 流动性
    if constraints_ok(S) break
end
return S

```

**关键要点**：

- **风险优先**：`max_risk` 优先；若不满足，编译器必须添加保护腿或拒绝生成。
- **成本次之**：`premium_offset` 是尽力满足项，绝不能违反 `max_risk`；若无法满足，输出警告 **W403**。
- 每次结构编辑后，系统整体 Greeks 会重新评估。

### 4.2 路径 B —— 从意图生成初稿，再全局微调 Greeks

```
S₀ ← one‑shot_generate(intent, constraints)
S₁ ← global_tweak(S₀, target_Greeks, max_risk, premium_offset)
S₂ ← reality_check(S₁)  # 资本效率、流动性、盈亏比
if !ok  then iterate or fallback(Path A)
return S₂

```

适用于希望“一步优化”而非交互式编辑的场景。

### 4.3 路径 C —— 反向解目标 Greeks（以市场响应为起点）

```
target_G ← user_or_model_vector()     # 目标 Δ,Γ,V,Θ,ρ …
candidate ← solve_leg_set(target_G, market, constraints)
# 再进入风险 & 资金管道
candidate ← ensure_max_risk(candidate, max_risk)
candidate ← apply_premium_offset(candidate, premium_offset)
return reality_check(candidate)

```

路径 C 计算量大（线性/二次规划或启发式搜索），适用于机器学习或机构引擎。

- **路径 A**：强调交互式、逐步构建。
- **路径 B**：批量构建后全局微调。
- **路径 C**：从目标 Greeks 向后反推构造（参考算法：Δ/Θ 软约束线性规划 + γ/V 贪心填充；复杂度 ≈ O(k·n²)）。

---

## 5 调整模块 [核心]

**用途**：当原始交易假设被削弱但尚未完全失效时，对已有仓位进行再配置。

| 字段 | 描述 |
| --- | --- |
| `adjust_when` | 布尔触发条件 —— 与 `entry_when` 使用相同语法。为真时，引擎尝试执行操作序列。 |
| `adjust_playbook` | 原子动作的有序列表（如 `roll`、`hedge`、`add_leg`、`remove_leg`、`delta_neutralise` 等）。按顺序执行，直到调整后满足风险约束。 |
| `abort_if` | 守护条件 —— 每个操作前评估。若为真，终止尝试并转入退出模块。 |

**示例**：

```
adjust_when  price_retrace(>-3%) or vega_crush(>20%)
adjust_playbook [ roll_nearest_term(), hedge_gamma(target=0.10) ]
abort_if additional_margin(%) > 15

```

---

## 6 退出模块 [核心]

**用途**：定义平仓或到期的终止条件。

| 字段 | 描述 |
| --- | --- |
| `exit_mode` | 多个触发条件的评估方式：`first_hit`（默认）、`all_required` 或 `machine_decide(score_fn)` |
| `exit_triggers` | 条件数组，例如 `price_target_hit(108)`、`pnl_percent(<‑25%)`、`days_left(<2d)`、`structure_destabilised(delta_jump)` 等 |

**结构模板**：

```
exit_mode first_hit
exit_triggers [
    price_target_hit(108),
    pnl_percent(>30%),
    days_left(<2d)
]

```

若未定义退出模块，引擎将发出警告并默认使用 `exit_triggers [ expiry_passed() ]`。

---

## 7 语法与验证 [核心]

### 7.1 EBNF 文法（文本形式）

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
Tenor         ::= TIME_LITERAL   (* 例如 30d, 14d *)
Expiry        ::= DATE_LITERAL   (* ISO‑8601 日期 *)
ArgList       ::= Expression (',' Expression)*

```

配套的 JSON 模式文件 `opl-lang‑1.0.schema.json` 用于严格机器校验。

### 7.2 验证码表

| 代码 | 含义 |
| --- | --- |
| `E101` | 未知原始组件类型 |
| `E201` | 禁用的杠杆 ETF 长周期腿 (§ 3.3) |
| `E301` | 超出资本限制 |
| `E302` | 构建完成后超出最大风险 |
| `E303` | 权利金抵消目标无法达成 |

**警告**：

| 代码 | 含义 |
| --- | --- |
| `W401` | 阻止 0DTE 且远 OTM 腿加入 |
| `W402` | 提前深度 ITM 腿：建议滚动/平仓 |
| `W403` | 权利金抵消目标被下调以满足最大风险限制 |

---

## 8 兼容性模板库 [可选]

### 8.1 等价 / 命名模式

| 别名 | 展开形式 |
| --- | --- |
| **Stock** | `Option(Call,K=P,T→∞) + Option(Put,K=P,T→∞)` |
| **Vertical(Call,K₁<K₂)** | `long call@K₁ + short call@K₂`（相同到期） |
| **Condor** | `VerticalCallBull + VerticalCallBear` |

### 8.2 结合性与规范化

```python
# normalise_strategy(S): 展平、排序、合并数量
legs = flatten_parentheses(S)
legs = merge_same_legs(legs)
legs = [l for l in legs if l.qty != 0]
legs = sort_by((underlying,strike,expiry,type,polarity))
return Σ legs

```

---

## 9 术语表 [核心]

| 术语 | 定义 |
| --- | --- |
| `C` | 原始组件 |
| `O` | 操作组件 |
| `S` | 策略系统 |
| `Δ, Γ, V, Θ, ρ, Charm` | 标准期权 Greeks 与一阶利率/时间敏感性 |
| `qty` | 合约数量（整数） |
| `polarity` | +1 多头 / ‑1 空头 |
| `intent` | Bull / Bear / Neutral / Not‑Bull / Not‑Bear |
| `vega_style` | Debit（多波动率） / Credit（空波动率） |

---

## 10 附录 A —— 启发式防护库（完整表）

（详见 §3.3，支持通过 `W4xx` 扩展）

---

## 11 附录 B —— 黄金示例 [可选]

```python
# 示例 1 多头垂直价差（DSL）
S = long Option(Call,480,14d) + short Option(Call,500,14d)

# 示例 2 合成股票（JSON）
[
  {"qty":1,"side":"long","type":"call","strike":480,"exp":3650},
  {"qty":1,"side":"short","type":"put","strike":480,"exp":3650}
]

# 示例 3 用语法糖表达的 3× 空头看跌
S = 3×short Option(Put,450,30d)

# 示例 4 Condor 的别名展开
S = Condor(Call, K1=480, K2=520, T=30d)

# 示例 5 根据权利金抵消目标自动转为借方垂直价差
intent           = Bull
premium_offset   = 40%
max_risk         = 1500 USD

# 编译器输出（示意）：
S = long Option(Call,480,30d) + short Option(Call,520,30d)

```

---

## 12 附录 C —— 语言集成流程（自然语言 ➜ 策略 ➜ 执行）

🧠「OPL‑Lang 的目标是成为人类意图与自动化策略引擎之间的接口语言。」

本节展示 OPL‑Lang 如何作为语义桥梁连接人类输入与机器执行。

```
人类自然语言意图
        ↓
AI 语义解析器（LLM / 规则 / few-shot 示例）
        ↓
OPL‑Lang 策略结构（意图 + 约束 + 组件）
        ↓
执行引擎（Python / Rust / C++）
        ↓
回测 / 实盘下单系统 / 投资组合模拟器

```

在该流程中：

- **人类** 表达高层意图（例如“我预期突破且波动率会上升，最大亏损低”）；
- **AI** 将其解析为 OPL‑Lang 结构，使用预定义映射或训练模型；
- **OPL‑Lang** 提供结构语法，完整表达方向、风险、周期、资金等；
- **执行引擎** 消化该结构并转换为实盘操作、回测或模拟。

---

## 13 版本控制与兼容性

OPL‑Lang 遵循 **语义化版本控制 2.0.0**：

- **主版本（MAJOR）**：破坏性语法变更（如 2.0）
- **次版本（MINOR）**：向后兼容的新特性（如 1.1）
- **修订版（PATCH）**：编辑性修正，不影响语法（如 1.0.1）

源码文件 **必须** 在前 256 字节内声明版本标识头：

```
opl-lang 1.0   # 版本标识后允许写注释

```

工具 **必须** 拒绝解析无法识别的主版本。

---

© 2025 OPL‑Lang Authors. MIT License. Version 1.0.0‑rc3（2025 年 5 月 20 日）

[← 返回目录主页](index.md)

