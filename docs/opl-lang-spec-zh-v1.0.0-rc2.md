[← Back to index](index.md)

📙 [点击查看中文版本 ➜ OPL‑Lang 规范 v1.0.0‑rc2](opl-lang-spec-zh-v1.0.0-rc2.md)

> **English version click here / 英文版本请点此**
> EN 📄 [OPL‑Lang White Paper – lean specification v1.0.0‑rc2](opl-lang-spec-en-v1.0.0-rc2.md)

# OPL‑Lang 规范 — 完整简体中文版 **v 1.0.0‑rc2**

> **状态** 发布候选版 2 — 2025‑05‑20
> **读者** 量化开发 / 编译器工程师 / API 实现者
> **许可证** MIT
> **非目标** 教学性内容、盈亏图（见将出版的交易员手册）

---

## 0 语言目标与范围 \[Core]

OPL‑Lang 是一门 **领域特定语言 (DSL)**，用于以同时 **机器可解析** 且 **人类可读写** 的形式来描述、生成并评估股票期权策略。它 **不是** 定价引擎；数值估值由下游模块完成。

**当前排除（v 1.x）**：奇异期权、多货币结算、加密资产、投资组合层级保证金净额；计划在 2.x 版本讨论。

* 带 **\[Core]** 标签的章节为规范性内容——实现 **必须 (MUST)** 遵守。
* **\[Optional]** 章节可选实现，忽略不会导致语言功能缺失。

---

## 1 核心语言组件 \[Core]

### 1.1 原始组件 `C`

| 符号                             | 描述                                           |
| ------------------------------ | -------------------------------------------- |
| `Cash`                         | 无风险占位资产，按短期利率 **r** 计息                       |
| `Stock`                        | 标的现货线性工具                                     |
| `Future`                       | 现金结算远期合约，到期 **T**；保证金规则由交易所或券商定义             |
| `ETF`                          | 一篮子线性工具                                      |
| `LeveragedETF{β}`              | *杠杆* (β×) 路径依赖工具，β ∈ ℝ⁺，默认 1；§ 3.3 对长周期合约有限制 |
| `Option(Call)` / `Option(Put)` | 标准化期权合约，执行价 **K**，到期 **T**                   |

### 1.2 操作组件 `O`

```text
O = qty n × polarity (+1│-1) × C
```

* `qty n` 正整数（默认 1）
* `polarity` +1 = **多头**，‑1 = **空头**
* **语法糖** `n×O(...)` 展开为 *n* 个相同持仓

### 1.3 策略系统 `S`

```text
S = Σ Oᵢ   (i = 1…N)
```

`S` 的净响应为各腿 Greeks 的 **非线性聚合**（见 § 2）。

---

## 2 系统响应模型 \[Core]

每个期权暴露六个一阶敏感度：

```text
Δ  (delta)   – ∂V/∂Price
Γ  (gamma)   – ∂Δ/∂Price
V  (vega)    – ∂V/∂IV
Θ  (theta)   – ∂V/∂Time
ρ  (rho)     – ∂V/∂Rate
Charm        – ∂Δ/∂Time
```

**局部可加性（一级）**†  在状态 `(S₀,t₀,σ₀,r₀)` 处，对策略 `S = Σ Vᵢ` 有：

```text
G_loc(S) := Σ G_loc(Vᵢ) ,  G ∈ {Δ, Γ, V, Θ, ρ, Charm}
```

实现 **应当 (SHOULD)** 在市场状态显著变化时重算 Greeks；高阶或路径依赖项由实现自定。

†“局部” 指该等式仅在无限小扰动下成立，不假设全局线性。

---

## 3 输入与约束层 \[Core]

### 3.1 市场输入

| 字段      | 含义              |
| ------- | --------------- |
| `price` | 现价 *P(t)*       |
| `iv`    | 隐含波动率曲面 *IV(t)* |
| `time`  | 时钟时间 *t*        |
| `rate`  | 无风险利率 *r(t)*    |

### 3.2 策略意图输入

| 字段                 | 取值 / 示例                             | 说明                     |
| ------------------ | ----------------------------------- | ---------------------- |
| `direction_intent` | Bull • Bear • RangeBound • Breakout | Breakout 表示预期大幅波动但方向不定 |
| `vol_intent`       | ↑IV • ↓IV • =IV                     | 预期波动方向                 |
| `vega_style`       | Debit • Credit                      | 区分 Neutral 是多波还是空波     |
| `holding_horizon`  | 整数天或 intraday / swing / multi-week  | 如 14                   |

> `direction_intent = Breakout` 时，编译器 **应当** 生成 Delta 中性、Gamma 高的起始结构（如多头跨式）。

资源与风控约束

| 字段                        | 含义                                                       |
| ------------------------- | -------------------------------------------------------- |
| `capital_limit`           | 最高名义金额 / 保证金                                             |
| `liquidity_floor`         | 最低持仓量 / 深度                                               |
| `granularity`             | **100** 股 / 合约（固定手数）                                     |
| `ban_leveraged_longdated` | **TRUE** — 禁止在 `LeveragedETF{β}` 上创建到期 **T > 45 天** 的期权腿 |

### 3.3 内置防护栏 \[Optional]

| 防护规则 DSL                                             | 理由                    | 默认动作                   |
| ---------------------------------------------------- | --------------------- | ---------------------- |
| `avoid_leg_if days_left < 2d && strike_offset > 10%` | 「0DTE 远 OTM」— 几乎无时间价值 | 阻止生成 / `W401`          |
| `warn_if days_left > 30d && leg_pnl > +25%`          | 长周期早早深 ITM，未实现利润易回吐   | 建议 roll / close；`W402` |
| `exit_triggers += days_left < 2d`                    | 保障临近到期仓位自动平仓          | 运行时追加触发                |

实现 **MAY** 增加更多规则（IV 崩溃、流动性枯竭等），使用 `W4xx` 命名空间。

### 3.4 方向到组件映射 \[Core]

| 意图 `I`           | 基础腿集合 `S(I)`                   |
| ---------------- | ------------------------------ |
| Bull             | { `long call`, `short put` }   |
| Bear             | { `long put`, `short call` }   |
| Neutral (Debit)  | { `long call` + `long put` }   |
| Neutral (Credit) | { `short call` + `short put` } |
| Not‑Bear         | Bull ∨ Neutral                 |
| Not‑Bull         | Bear ∨ Neutral                 |

**规范化**：`Breakout` → `Neutral (Debit)`；`RangeBound` → `Neutral (Credit)`（先规范化再执行 `MapIntentToO`）。\`MapIntentToO(I, vega\_style)\` \*\*可以\*\* 返回一条或多条腿；编译器 \*\*必须\*\* 将所有返回的腿注入为路径 A/B/C 的起始集合。

## 4 策略构建路径 \[Core]

### 4.1 路径 A — 逐步构建（交互式）

```text
base ← MapIntentToO(intent, vega_style)
S    ← {base}
loop
    adjust_or_add_leg(S, next_constraint)
    G ← EvaluateGreeks(S)
    if constraints_ok(S) then break
end
return S
```

### 4.2 路径 B — 批量构建 + 全局优化

```text
candidate ← GenerateByIntentAndMarket(intent, market, constraints)
G         ← EvaluateGreeks(candidate)
candidate ← GlobalTweak(candidate, G, constraints)
return candidate
```

### 4.3 路径 C — 目标 Greeks 反求

```text
target_G  ← UserOrModelGreeksTarget()
candidate ← SolveForLegSet(target_G, market, constraints)
return EnforceConstraints(candidate)
```

*路径 A* 强调交互式逐步构建；*路径 B* 批量生成后进行全局微调；*路径 C* 从目标 Greeks 向后求解，采用启发式搜索（参考算法：对 Δ/Θ 软约束做线性规划，然后贪婪填充 γ/vega，复杂度约 O(k·n²)）。

## 5 调整模块 \[Core]

| 字段                | 描述                                     |
| ----------------- | -------------------------------------- |
| `adjust_when`     | 布尔条件，触发调整                              |
| `adjust_playbook` | 原子动作序列 (`roll`, `hedge`, `add_leg`, …) |
| `abort_if`        | 守卫条件；若为真则终止调整转至退出                      |

示例：

```text
adjust_when  price_retrace(>-3%) or vega_crush(>20%)
adjust_playbook [ roll_nearest_term(), hedge_gamma(target=0.10) ]
abort_if additional_margin(%) > 15
```

---

## 6 退出模块 \[Core]

定义平仓或到期条件。

| 字段              | 描述                                                                     |
| --------------- | ---------------------------------------------------------------------- |
| `exit_mode`     | 多触发器评估策略：`first_hit`(默认) / `all_required` / `machine_decide(score_fn)` |
| `exit_triggers` | 触发条件数组，如 `price_target_hit(108)`、`pnl_percent(<-25%)`、`days_left(<2d)` |

```
exit_mode first_hit
exit_triggers [
    price_target_hit(108),
    pnl_percent(>30%),
    days_left(<2d)
]


```

*示例骨架*

如未声明退出模块，系统警告并默认 `exit_triggers [ expiry_passed() ]`。

---

## 7 语法与校验 \[Core]

### 7.1 EBNF 语法

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
Tenor         ::= TIME_LITERAL
Expiry        ::= DATE_LITERAL
ArgList       ::= Expression (',' Expression)*
```

### 7.2 错误代码

| 代码     | 含义                    |
| ------ | --------------------- |
| `E101` | 未知原始类型                |
| `E201` | 杠杆 ETF 长周期腿被禁 (§ 3.3) |
| `E301` | 超出资金上限                |

---

## 8 兼容模板库 \[Optional]

| 别名                        | 展开                                           |
| ------------------------- | -------------------------------------------- |
| **Stock**                 | `Option(Call,K=P,T→∞) + Option(Put,K=P,T→∞)` |
| **Vertical(Call,K₁\<K₂)** | `long call@K₁ + short call@K₂` (同到期)         |
| **Condor**                | `VerticalCallBull + VerticalCallBear`        |

### 8.2 关联与规范化

```text
legs = flatten_parentheses(S)
legs = merge_same_legs(legs)
legs = filter(qty≠0)
legs = sort_by((underlying,strike,expiry,type,polarity))
return Σ legs
```

---

## 9 术语表 \[Core]

| 术语                     | 定义                                          |
| ---------------------- | ------------------------------------------- |
| `C`                    | 原始组件                                        |
| `O`                    | 操作组件 / 单腿                                   |
| `S`                    | 策略系统                                        |
| `Δ, Γ, V, Θ, ρ, Charm` | 标准 Greeks 与相关一阶敏感度                          |
| `qty`                  | 合约乘数（整数）                                    |
| `polarity`             | +1 多头 / -1 空头                               |
| `intent`               | Bull / Bear / Neutral / Not-Bull / Not-Bear |
| `vega_style`           | Debit（多波）/ Credit（空波）                       |

---

## 10 附录 A — 防护规则库

完整守护规则同 § 3.3 表，无差异。

---

## 11 附录 B — 黄金示例

```text
# 1 牛市垂直价差
S = long Option(Call,480,14d) + short Option(Call,500,14d)

# 2 合成现货 (JSON)
[
  {"qty":1,"side":"long","type":"call","strike":480,"exp":3650},
  {"qty":1,"side":"short","type":"put","strike":480,"exp":3650}
]

# 3 三份卖认沽
S = 3×short Option(Put,450,30d)

# 4 Condor 别名
S = Condor(Call, K1=480, K2=520, T=30d)
```

---

## 12 版本与兼容性

**语义化版本 2.0.0**（MAJOR / MINOR / PATCH）。文件需以魔术头开头：

```text
opl-lang 1.0   # 可附注释

```

工具 **MUST** 拒绝解析无法识别主版本号的文件。

© 2025 OPL‑Lang Authors. MIT License. 版本 1.0.0‑rc2 (2025-05-20)

[← 返回目录主页](index.md)

