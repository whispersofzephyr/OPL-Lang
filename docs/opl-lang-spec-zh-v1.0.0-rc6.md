# OPL‑Lang 白皮书 – 精简规范 **v 1.0.0‑rc6**

> 状态 发布候选版本 6 — 2025年5月24日
> **目标读者** 量化工程师 / 编译器开发者 / API 实现者
> **许可证** MIT
> **非目标** 教学用途、盈亏图示（请参见即将发布的交易者手册）

[← 返回索引](index.md) | 📙 [Click to view English version ➜ OPL‑Lang Spec v1.0.0‑rc6](opl-lang-spec-en-v1.0.0-rc6.md)

---

## 0 语言目标与适用范围

**OPL‑Lang** 是一个用于 *描述、生成与评估* 股票 + 期权混合策略的 **领域专用语言（DSL）**，同时兼具 **可供机器解析** 和 **人类可书写性**。它 **不是** 一个定价引擎；数值定价的职责由下游组件承担。

*1.x 版本线不包含的内容：* 奇异期权 / 亚洲期权 / 障碍期权、多币种结算、加密资产标的、跨多个标的的差异化交易，以及经纪商特定的投资组合保证金聚合。

* **\[核心]** 部分具备规范性 —— 实现必须 **MUST** 遵循。
* **\[可选]** 部分可被忽略而不会影响语言完整性。

---

## 1 核心语言组成 \[核心]

### 1.1 基础组件 `C`

| 符号                             | 描述                                         |
| ------------------------------ | ------------------------------------------ |
| `Cash` *(可选)*                  | 无风险占位符；按短期利率 *r* 计息。引擎 **可拒绝** 使用空头现金。     |
| `Stock`                        | 标的资产上的线性工具（股票）。                            |
| `Future`                       | 线性远期合约，到期时间为 **T**；适用交易所/券商保证金规则。          |
| `ETF`                          | 一篮子资产的线性工具。                                |
| `LeveragedETF{β}` *(可选)*       | 杠杆（β 倍）路径相关工具，β ∈ ℝ⁺，默认值为 1。长期持有受限于 § 3.3。 |
| `Option(Call)` / `Option(Put)` | 标准化合约，具备执行价 **K** 与到期时间 **T**。             |

> 标记为 **可选** 的行可以被轻量级引擎忽略。

### 1.2 操作组件 `O`

```opl
O = qty n × polarity (+1│-1) × C
```

* **`qty n`** 正整数（默认值为 1）。控制模块可以通过 `scale_qty` 调整 n。
  **合约乘数说明：** *1 份期权合约 = 100 股标的资产*。引擎 **必须** 依据此常量（或本地交易所规范）转换名义本金。
* **`polarity`** +1 表示多头，‑1 表示空头。
* **语法糖** `n×O(...)` 展开为 *n* 个完全相同的腿。

### 1.3 策略系统 `S`

```opl
S = Σ Oᵢ   (i = 1…N)
```

`S` 以腿的希腊值 **非线性聚合** 方式响应。

---

## 2 系统响应模型 \[核心]

```
Δ (delta)   – ∂V/∂Price
Γ (gamma)   – ∂Δ/∂Price
V (vega)    – ∂V/∂IV
Θ (theta)   – ∂V/∂Time
ρ (rho)     – ∂V/∂Rate
Charm       – ∂Δ/∂Time
```

对于策略 `S = Σ Vᵢ` 在状态 `(S₀,t₀,σ₀,r₀)` 下：

```opl
G_loc(S) := Σ G_loc(Vᵢ)     G ∈ {Δ, Γ, V, Θ, ρ, Charm}
```

* 实现 **必须** 提供准确的 Δ、Γ、V、Θ。
* 实现 **应当** 提供 ρ 与 Charm（若到期日大于 45 天或需考虑利率敏感性 / Delta 对冲自动化），否则可以返回 `null`。
* 实现 **应当** 在市场状态显著变化时重新计算希腊值。

> **线性腿说明：** `Stock` / `ETF` / `Future` 提供 Δ = ±1（根据方向），其余希腊值为 0。
>
> **控制变量与希腊值关系：** 修改 {执行价 *K*、到期 *T*、腿集合 *L*、数量 *Q*} 会重新加权组合希腊值；引擎可以直接在这些控制项上求解目标希腊值。

---

## 3 输入与约束层 \[核心]

### 3.1 市场输入

| 字段      | 含义           |
| ------- | ------------ |
| `price` | 即时现价 *P(t)*  |
| `iv`    | 隐含波动率曲面      |
| `time`  | 当前时间 *t*     |
| `rate`  | 无风险利率 *r(t)* |

### 3.2 意图、风险与资金约束

| 字段                      | 示例取值 / 域                            | 角色说明                            |
| ----------------------- | ----------------------------------- | ------------------------------- |
| `direction_intent`      | Bull • Bear • RangeBound • Breakout | 主要方向假设                          |
| `vol_intent` *(可选)*     | ↑IV • ↓IV • =IV                     | 明确的波动率观点；如省略，则由 `vega_style` 推断 |
| `vega_style`            | Debit • Credit                      | 区分中性策略是多波动率还是空波动率               |
| `holding_horizon`       | 10天 / 日内 / 波段                       | 持仓周期                            |
| `price_target` *(可选)*   | 120 USD                             | 目标价格点（用于退出参考）                   |
| `vol_target` *(可选)*     | 35 IV                               | 目标波动率水平                         |
| `max_risk` *(可选)*       | 2,000 USD                           | 最差情况下的最大亏损上限                    |
| `premium_offset` *(可选)* | 50%                                 | 期权支出希望通过信用腿抵消的比例                |

### 3.3 资源与流动性约束

| 字段                        | 含义                                      |
| ------------------------- | --------------------------------------- |
| `capital_limit`           | 可接受的最大名义本金 / 保证金                        |
| `liquidity_floor`         | 所需最小持仓量 / 深度                            |
| `granularity`             | 100 股 / 合约为基本交易单位                       |
| `ban_leveraged_longdated` | TRUE 表示禁止使用到期大于 45 天的 `LeveragedETF{β}` |

### 3.4 内建启发式守护逻辑 \[可选]

守护逻辑默认启用；如需禁用某项，则需加入 `--no-guard XYZ`。

| 守护规则 DSL                                         | 逻辑解释                | 默认行为                       |
| ------------------------------------------------ | ------------------- | -------------------------- |
| `avoid_leg_if days_left<2d && strike_offset>10%` | “0DTE 且深度 OTM”，价值极低 | 阻止生成；发出警告 **W401**         |
| `warn_if days_left>30d && leg_pnl>+25%`          | 过早深 ITM 盈利易被回撤      | 提示进行 `roll` 或 `close`；W402 |
| `exit_triggers += days_left<2d`                  | 临近到期自动平仓            | 在运行时追加触发器                  |

引擎 **可以** 添加自定义的守护（如波动骤降、流动性枯竭），使用 `W4xx` 编码。

### 3.5 方向意图到组件映射 \[核心]

| 意图类型             | 允许的基础组件集合 `S(I)`                            |
| ---------------- | ------------------------------------------- |
| Bull             | { `long Stock`, `long call`, `short put` }  |
| Bear             | { `short Stock`, `long put`, `short call` } |
| Neutral (Debit)  | { `long call` + `long put` }                |
| Neutral (Credit) | { `short call` + `short put` }              |
| Not‑Bear         | Bull ∨ Neutral                              |
| Not‑Bull         | Bear ∨ Neutral                              |

**规范化规则：** `Breakout` → `Neutral (Debit)`；`RangeBound` → `Neutral (Credit)`。

`MapIntentToO` 在无需波动敞口时 **可以** 返回线性组件（如 `Stock` / `Future`）。

---

## 4 策略构建路径 \[核心]

### 4.0 控制模块

| 符号    | 单腿场景      | 多腿场景 (N ≥ 1)       |
| ----- | --------- | ------------------ |
| **K** | 执行价       | 向量 **K** = (K₁…Kₙ) |
| **T** | 到期时间      | 向量 **T** = (T₁…Tₙ) |
| **L** | 腿集合（固定长度） | {O₁…Oₙ}            |
| **Q** | 数量        | 向量 **Q** = (q₁…qₙ) |

*对于线性腿，仅适用 **`scale_qty`**, **`add_leg`**, **`remove_leg`**；**`shift_strike`** 与 **`roll_tenor`** 为无操作（NOP）。*

调整 {K, T, L, Q} 与插入、移除、缩放腿等价，均会影响组合的希腊值。

### 原子控制操作

| 操作                             | 参数类型    | 效果说明           |
| ------------------------------ | ------- | -------------- |
| `shift_strike(leg=i, ΔK)`      | 绝对值或百分比 | 调整第 *i* 腿的执行价  |
| `roll_tenor(leg=i, ΔT)`        | 天数      | 移动第 *i* 腿的到期时间 |
| `replace_leg(i, newO)`         | —       | 用新腿替换第 *i* 腿   |
| `add_leg(O)` / `remove_leg(i)` | —       | 扩展或收缩腿集合       |
| `scale_qty(leg=i, factor)`     | 浮点数     | 乘以数量系数 *qᵢ*    |

上述控制动词适用于每条路径循环与调整模块中。

### 4.1 路径 A — 意图驱动的逻辑树

**依赖链：**

```
市场快照
   ↓
方向 / 波动意图
   ↓   → 初始腿选择
市场反馈 + 控制调整
   ↓   → 细化希腊值
风险限制检查
   ↓
应用成本抵消比例（如指定）
   ↓
资源 / 流动性验证
   ↓
循环直至满足全部约束
```

> 最后进行资源检查，以防止中间变更引发误判。

*重点：*

* **优先风险控制：** `max_risk` 优先级最高，若超限必须添加保护翼或直接拒绝。
* **其次考虑成本：** `premium_offset` 是最佳努力，不能突破风险上限。
* 控制操作可在任意循环中应用。
* 每次结构更改后都会重新计算系统希腊值。

### 4.2 路径 B — 批量生成 → 全局微调

```
S₀ ← one_shot_generate(intent, constraints_with_basic_liquidity)  # 轻量预筛选
S₁ ← global_tweak(S₀, target_Greeks, max_risk, premium_offset)
S₂ ← reality_check(S₁)    # 完整的资本与流动性验证
if !ok  then iterate or fallback(Path A)
return S₂
```

* 用于偏好单次优化流程而非交互式编辑的场景。

### 4.3 路径 C — 希腊值目标反向求解（先反应市场）

```
target_G ← user_or_model_vector()     # 期望的 Δ, Γ, V, Θ, ρ …
candidate ← solve_leg_set(target_G, market, constraints)
── control_tune(candidate, policy)     # strike/tenor/leg/qty 微调
candidate ← ensure_max_risk(candidate, max_risk)
candidate ← apply_premium_offset(candidate, premium_offset)

return reality_check(candidate)
```

求解完成后，引擎 **可以** 调用 `control_tune` 进行最终微调，然后执行风险与成本校验。

路径 C 计算开销较大（可能采用线性/二次规划或启发式搜索），适合机器学习 / 机构引擎。

---

## 5 调整模块 \[核心]

**用途：** 当原始观点有所削弱但未完全失效时，对已有持仓进行再设计。

| 字段                | 描述说明                                                                                       |
| ----------------- | ------------------------------------------------------------------------------------------ |
| `adjust_when`     | 布尔触发条件，与 `entry_when` 使用相同语法。为真时引擎尝试执行调整脚本。                                                |
| `adjust_playbook` | 原子操作组成的有序列表（如 `roll`, `hedge`, `add_leg`, `remove_leg`, `scale_qty`, `delta_neutralise` 等） |
| `abort_if`        | 守护条件 —— 每条操作前判定；为真时中止调整，转入退出模块。                                                            |

*示例：*

```
adjust_when  price_retrace(>-3%) or vega_crush(>20%)
adjust_playbook [ roll_nearest_term(), hedge_gamma(target=0.10) ]
abort_if additional_margin(%) > 15
```

---

## 6 退出模块 \[核心]

**用途：** 定义平仓或自然到期的触发条件。

| 字段              | 描述说明                                                                     |
| --------------- | ------------------------------------------------------------------------ |
| `exit_mode`     | 多个触发器的评估方式：`first_hit`（默认）/ `all_required` / `machine_decide(score_fn)`  |
| `exit_triggers` | 条件数组，如：`price_target_hit(108)`, `pnl_percent(<‑25%)`, `days_left(<2d)` 等 |

*结构模板：*

```
exit_mode first_hit
exit_triggers [
    price_target_hit(108),
    pnl_percent(>30%),
    days_left(<2d)
]
```

若未定义退出模块，引擎将发出 **警告**，并默认使用 `exit_triggers [ expiry_passed() ]`。

---

## 7 文法与验证 \[核心]

### 7.1 EBNF 文法（文本形式）

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
Strike        ::= NUMBER | RelNUMBER
Tenor         ::= TIME_LITERAL | RelTIME
Expiry        ::= DATE_LITERAL
RelNUMBER     ::= ('+'|'-') NUMBER '%'
RelTIME       ::= ('+'|'-') NUMBER 'd'
ArgList       ::= Expression (',' Expression)*
```

官方 JSON 模式（`opl-lang‑1.0.schema.json`）随参考实现发布，用于严格的机器校验。

### 7.2 验证码

| 代码     | 含义说明                    |
| ------ | ----------------------- |
| `E101` | 未知的原始类型                 |
| `E201` | 禁用的杠杆 ETF 长期腿 (§ 3.3)   |
| `E301` | 超出资本限制                  |
| `E302` | 最终生成后仍超出 `max_risk` 限制  |
| `E303` | `premium_offset` 目标无法满足 |
| `E304` | 缩放后数量低于最小单位             |
| `E305` | 检测到负数或 0 数量             |

**警告码：**

| 代码     | 含义说明                                |
| ------ | ----------------------------------- |
| `W401` | 阻止 0DTE 且远 OTM 的腿生成                 |
| `W402` | 深度 ITM 腿过早盈利，建议回滚或平仓                |
| `W403` | 为满足 `max_risk`，降低了 `premium_offset` |

---

## 8 兼容模板库 \[可选]

### 8.1 等价表达式 / 命名模式

| 别名                        | 规范化展开式                                       |
| ------------------------- | -------------------------------------------- |
| **Stock**                 | `Option(Call,K=P,T→∞) + Option(Put,K=P,T→∞)` |
| **Vertical(Call,K₁\<K₂)** | `long call@K₁ + short call@K₂`（相同到期日）        |
| **Condor**                | `VerticalCallBull + VerticalCallBear`        |

### 8.2 结合律与规范化流程

```
# normalise_strategy(S): 扁平化、排序、合并数量
legs = flatten_parentheses(S)
legs = merge_same_legs(legs)
legs = [l for l in legs if l.qty != 0]
legs = sort_by((underlying,strike,expiry,type,polarity))
return Σ legs
```

## 9 术语表 \[核心]

| 术语                     | 定义说明                                        |
| ---------------------- | ------------------------------------------- |
| `C`                    | 基础组件（Cash / Stock / Option 等）               |
| `O`                    | 操作组件（数量 × 方向 × 组件）                          |
| `S`                    | 策略系统，由多个 O 组成                               |
| `K`                    | 执行价（或向量）                                    |
| `T`                    | 到期日 / 持有期（或向量）                              |
| `L`                    | 腿集合（Leg Set）                                |
| `Q`                    | 数量（或向量）                                     |
| `Δ, Γ, V, Θ, ρ, Charm` | 标准期权希腊值及一阶利率/时间敏感度                          |
| `qty`                  | 整数型合约倍数                                     |
| `polarity`             | +1 多头 / ‑1 空头                               |
| `intent`               | Bull / Bear / Neutral / Not‑Bull / Not‑Bear |
| `vega_style`           | Debit（多波动率）/ Credit（空波动率）                   |
| *Linear leg*           | 指 Stock / ETF / Future，Δ = ±1（带符号），其余希腊值为 0 |

---

## 10 附录 A — 启发式守护库（完整表）

（见 § 3.3 表格结构；本库可扩展，统一采用 W4xx 编号）

---

## 11 附录 B — 黄金样例 \[可选]

```
# 1 牛市垂直价差（DSL 表达）
S = long Option(Call,480,14d) + short Option(Call,500,14d)

# 2 合成多头股票（JSON 格式）
[
  {"qty":1,"side":"long","type":"call","strike":480,"exp":3650},
  {"qty":1,"side":"short","type":"put","strike":480,"exp":3650}
]

# 3 使用语法糖的 3× 空头看跌期权
S = 3×short Option(Put,450,30d)

# 4 使用别名生成 Condor
S = Condor(Call, K1=480, K2=520, T=30d)

# 5 设置 premium_offset = 40% 时，long call 自动转为垂直价差
intent         = Bull
premium_offset = 40%
max_risk       = 1500 USD

# 编译器输出（示意）
S = long Option(Call,480,30d) + short Option(Call,520,30d)

# 6 数量缩放示例
S = long Option(Call,480,30d)
adjust_playbook [ scale_qty(leg=1, factor=2.0) ]  # 名义值翻倍

# 7 备兑开仓
S = long Stock + short Option(Call,520,30d)
```

---

## 12 附录 C — 语言集成流程（自然语言 ➜ 策略结构 ➜ 执行）

🧠 **“OPL‑Lang 的目标是成为人类意图与自动策略引擎之间的接口语言。”**

```
人类自然语言意图
          ↓
AI 语义解析器（LLM / 规则 / 示例驱动）
          ↓
OPL‑Lang 策略结构（意图 + 约束 + 组件）
          ↓
执行引擎（Python / Rust / C++）
          ↓
回测 / 实盘下单系统 / 投资组合模拟器
```

在此流程中：

* **人类** 表达高层意图（例如“我预期上涨并伴随波动率上升，但最大亏损需受限”）；
* **AI** 将意图解析为 OPL-Lang 结构，可采用映射规则或模型学习；
* **OPL-Lang** 提供完整的结构语法，明确方向、风险、持有周期、资本等；
* **执行引擎** 消化该结构并转化为回测、实盘交易或模拟器任务。

---

## 13 版本控制与兼容性

OPL‑Lang 遵循 **语义版本规范 2.0.0**：

* **MAJOR**：不兼容的语法变更（如 2.0）
* **MINOR**：向后兼容的新功能（如 1.1）
* **PATCH**：编辑修复，无语法变动（如 1.0.1）

源文件需在前 256 字节内包含如下魔法头：

```
opl-lang 1.0   # 后续允许注释内容
```

工具必须拒绝解析无法识别主版本号的文件。

---

© 2025 OPL‑Lang Authors. MIT License. 版本 1.0.0‑rc6（2025年5月24日）

[← 返回索引](index.md)
