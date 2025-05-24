opl-lang 1.0   # magic header for files

**For Chinese version click here** / **中文版本请点击这里**  
ZH 📄 [OPL-Lang 规范 – 精简版 v1.0.0-rc2](opl-lang-spec-zh-v1.0.0-rc2.md)

# OPL-Lang White Paper – lean specification **v 1.0.0-rc2**

> **Status**  Release Candidate 2 – 20 May 2025  
> **Audience**  quant / compiler / API implementers  
> **License**  MIT  
> **Non-Goals**  pedagogy, payoff diagrams (see forthcoming trader book)

---

opl-lang 1.0   # magic header for files

> **For English version click here** / **英文版本请点击这里**
> EN 📄 [OPL‑Lang White Paper – lean specification v1.0.0‑rc2](opl-lang-spec-en-v1.0.0-rc2.md)

# OPL‑Lang 规范 — 精简版 **v 1.0.0‑rc2** （简体中文）

> **状态**   发布候选版 2 — 2025‑05‑20
> **适用读者**   量化/编译器/接口实现者
> **许可证**   MIT
> **非目标**   估值引擎（定价由下游模块完成）

---

## 0 语言目标、范围 \[Core]

OPL‑Lang 是一门 **领域特定语言 (DSL)**，用于以既 **机器可解析** 又 **人类可读写** 的形式描述、生成并评估股票期权策略。它 **不是** 估值引擎；数值定价交由下游插件完成。

**当前支持标的（v 1.0）**：美国上市股票与指数期权。
*未来工作*：加密货币、外汇、大宗商品等标的。

* 带有 **\[Core]** 标记的章节为规范性内容——实现 **必须 (MUST)** 遵守。
* **\[Optional]** 章节可选实现，忽略不会导致语言不完整。

---

## 1 核心语言组件 \[Core]

### 1.1 原始组件 `C`

| 符号                             | 描述                                           |
| ------------------------------ | -------------------------------------------- |
| `Cash`                         | 无风险占位资产，按短期利率 **r** 计息                       |
| `Stock`                        | 标的现货线性工具                                     |
| `Future`                       | 线性远期合约，到期 **T**；保证金规则由交易所/券商决定               |
| `ETF`                          | 一篮子线性工具                                      |
| `LeveragedETF{β}`              | *杠杆* (β×) 路径依赖工具，β ∈ ℝ⁺，默认 1；§ 3.3 对长周期合约有限制 |
| `Option(Call)` / `Option(Put)` | 标准化合约，执行价 **K**，到期 **T**                     |

> **注**：加入 `Cash` 与 `Future` 可方便表达换仓/箱体等合成结构。

### 1.2 操作组件 `O`

```text
O = qty n × polarity (+1│-1) × C
```

* `qty n` 正整数（默认 1）
* `polarity` +1 = **多头**，-1 = **空头**
* **语法糖** `n×O(...)` 展开为 *n* 个同样持仓

### 1.3 策略系统 `S`

```text
S = Σ Oᵢ   (i = 1…N)
```

`S` 的净响应为各腿 Greeks 的 **非线性聚合**（见 § 2）。

---

## 2 系统响应模型 \[Core]

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

†“局部” 指该等式仅在无限小扰动下成立，并不假设全局线性。

---

## 3 输入与约束层 \[Core]

### 3.1 市场输入

| 字段      | 含义              |
| ------- | --------------- |
| `price` | 现价 *P(t)*       |
| `iv`    | 隐含波动率曲面 *IV(t)* |
| `time`  | 时钟时间 *t*        |
| `rate`  | 无风险利率 *r(t)*    |

### 3.2 策略意图输入

| 字段                 | 取值 / 示例                             | 说明                        |
| ------------------ | ----------------------------------- | ------------------------- |
| `direction_intent` | Bull • Bear • RangeBound • Breakout | “Breakout” 指预期大幅波动但方向不定   |
| `vol_intent`       | ↑IV • ↓IV • =IV                     | 预期波动方向                    |
| `vega_style`       | Debit • Credit                      | 可选，用于区分 `Neutral` 是多波还是空波 |
| `holding_horizon`  | 整数天或 intraday / swing / multi-week  | 例：14                      |

> 当 `direction_intent = Breakout` 时，编译器 **应当** 默认生成一个无 Delta、高 Gamma 的起始结构（如多头跨式）。

资源与风控约束

| 字段                        | 含义                                                       |
| ------------------------- | -------------------------------------------------------- |
| `capital_limit`           | 最高名义金额 / 保证金                                             |
| `liquidity_floor`         | 最低持仓量 / 深度                                               |
| `granularity`             | **100** 股 • 每份合约 (固定手数)                                  |
| `ban_leveraged_longdated` | **TRUE** — 禁止在 `LeveragedETF{β}` 上创建到期 **T > 45 天** 的期权腿 |

### 3.3 内置防护栏 \[Optional]

（同英文版守护规则，表略）

### 3.4 方向到组件映射 \[Core]

**规范化规则**：`Breakout` → `Neutral (Debit)`；`RangeBound` → `Neutral (Credit)`（在调用 `MapIntentToO` **之前** 应用）。

| 意图 `I`           | 允许的基础腿集合 `S(I)`                |
| ---------------- | ------------------------------ |
| Bull             | { `long call`, `short put` }   |
| Bear             | { `long put`, `short call` }   |
| Neutral (Debit)  | { `long call` + `long put` }   |
| Neutral (Credit) | { `short call` + `short put` } |
| Not‑Bear         | Bull ∨ Neutral                 |
| Not‑Bull         | Bear ∨ Neutral                 |

`MapIntentToO(I, vega_style)` **可** 返回一条或多条腿；编译器 **必须** 注入全部返回腿作为路径 A/B/C 的起始集合。

---

## 4 策略构建路径 \[Core]

（保持与英文版一致，算法伪代码省略）

---

## 5 调整模块 \[Core]

（中文翻译，同英文版逻辑）

---

## 6 退出模块 \[Core]

（中文翻译，同英文版逻辑）

---

## 7 语法与校验 \[Core]

### 7.1 EBNF 语法

（保持英文符号与代码块不变，文本说明中文）

### 7.2 错误代码

| 代码     | 含义                    |
| ------ | --------------------- |
| `E101` | 未知原始类型                |
| `E201` | 杠杆 ETF 长周期腿被禁 (§ 3.3) |
| `E301` | 超出资金上限                |

---

## 8 兼容性模板库 \[Optional]

（表与英文相同，中文补注）

---

## 9 术语表 \[Core]

| 术语                     | 定义                                          |
| ---------------------- | ------------------------------------------- |
| `C`                    | 原始组件                                        |
| `O`                    | 操作组件 / 单腿                                   |
| `S`                    | 策略系统                                        |
| `Δ, Γ, V, Θ, ρ, Charm` | 标准期权 Greeks 及相关一阶敏感度                        |
| `qty`                  | 合约乘数（整数）                                    |
| `polarity`             | +1 多头 / -1 空头                               |
| `intent`               | Bull / Bear / Neutral / Not-Bull / Not-Bear |
| `vega_style`           | Debit（多波）/ Credit（空波）                       |

---

## 附录 A — 防护规则库

（同英文版）

## 附录 B — 黄金示例

```text
# 1  牛市价差 (DSL)
S = long Option(Call,480,14d) + short Option(Call,500,14d)

# 2  合成现货 (JSON)
[
  {"qty":1,"side":"long","type":"call","strike":480,"exp":3650},
  {"qty":1,"side":"short","type":"put","strike":480,"exp":3650}
]

# 3  批量卖平值认沽
S = 3×short Option(Put,450,30d)

# 4  通过别名快速写 Condor
S = Condor(Call, K1=480, K2=520, T=30d)
```

---

## 版本与兼容性

OPL‑Lang 遵循 **语义化版本 2.0.0**：MAJOR / MINOR / PATCH 规则同英文版。

文件 **必须** 以版本魔术头开头：

```text
opl-lang 1.0   # 可附注释
```

---

© 2025 OPL‑Lang Authors. MIT License. 版本 1.0.0‑rc2  (2025‑05‑20)
