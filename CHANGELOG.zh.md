# 📌 版本更新日志 – OPL‑Lang v1.0.0‑rc6

**发布日期**：2025‑05‑24  
**对比版本**：[v1.0.0‑rc3](docs/opl-lang-spec-en-v1.0.0-rc3.md)

---

## 🔧 结构与语义修订

- 明确 DSL 适用于混合 **股票 + 期权** 策略。
- 强化“不含定价引擎”这一定位说明。
- 各章节标题与语法区块格式更加统一规范。

## ➕ 新增功能与输入项扩展

- 新增输入字段：`price_target` 和 `vol_target`。
- 支持 **相对行权价与期限表达**（如 `+5%`、`‑30d`）。
- 补充术语表：`K`、`T`、`L`、`Q` 等控制参数。

## ⚙️ 引擎行为与约束机制优化

- 明确线性腿（Stock/ETF/Future）仅贡献 Δ = ±1，其他希腊值为 0。
- 强制引擎处理期权合约乘数（1 张合约 = 100 股）。
- 明确 `shift_strike` 和 `roll_tenor` 对线性腿为 NOP。
- 当不需要 vega 暴露时，`MapIntentToO` 可返回线性工具。

## 🧠 控制模块与风险逻辑更新

- `Path C` 添加 `control_tune()` 后处理步骤（以前仅隐含存在）。
- `Breakout` / `RangeBound` 意图自动标准化（canonicalisation rule）。
- 强化 `max_risk` 优先于 `premium_offset` 的编译器行为，并说明 `W403` 警告。

## 🧪 示例策略拓展

- 新增两个黄金示例：
  - 使用 `scale_qty()` 加倍头寸。
  - 标准 **备兑开仓（covered call）** 策略。

## 📐 语法规则与验证

- 新增相对表达的文法规则：`RelNUMBER` 与 `RelTIME`。
- 新增验证代码：
  - `E304`（不满足最小单位限制）
  - `E305`（数量为零或负）
  - `W403`（premium offset 降级）

---

**查看完整规范**：[opl-lang-spec-en-v1.0.0-rc6.md](docs/opl-lang-spec-en-v1.0.0-rc6.md)  
**上一个版本**：[v1.0.0-rc3](docs/opl-lang-spec-en-v1.0.0-rc3.md)
