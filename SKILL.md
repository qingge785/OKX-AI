---
name: BTC 4x 全仓 OrderBlock 策略（1小时K线测试版）
description: 只交易 BTC-USDT-SWAP 的 4x 全仓 Order Block + Pivot Points 策略，使用1小时K线识别支撑阻力，每15分钟巡查一次，模拟盘测试专用。Order Block 区间扩大 ±1.8%。
version: 1.2
author: qingge785
---

# 模拟盘 · 4x全仓 BTC Order Block + Pivot Points（1小时K线测试版） V1.2（Demo模式）

# 执行节奏
每 15 分钟自动触发一次

# Step 0 · 准备工作
1. 调用 swap_set_leverage 把 BTC-USDT-SWAP 杠杆设为 4，模式 cross
2. 调用 swap_get_positions 检查是否有 BTC 持仓，若有则跳过本轮
3. 调用 account_balance 查询可用余额

# Step 1 · 行情数据采集（仅 BTC）
- 用 market_get_candles 获取 BTC-USDT-SWAP 的 **1小时 K 线最近 120 根**（用于 Order Block 和 Pivot Points）
- 用 market_get_candles 获取 BTC-USDT-SWAP 的 **15分钟 K 线最近 40 根**（用于 3 连阳/阴判断）

# Step 2 · 支撑压力位识别（1小时K线 + 宽区间）
AI 请严格按以下规则识别：

**Order Block（主要信号，扩大区间）**：
- 看涨 Order Block（多头支撑）：最近一次强上涨前的最后一根明显阴线实体低点，向上下各扩展 **1.8%** 作为支撑区间
- 看跌 Order Block（空头压力）：最近一次强下跌前的最后一根明显阳线实体高点，向上下各扩展 **1.8%** 作为压力区间
- 只取最近 1-2 个**未被完全突破**的 Order Block

**Pivot Points（辅助确认）**：
用最近一根 **1小时** K 线计算标准 Pivot：
- Pivot = (High + Low + Close) / 3
- R1 = 2×Pivot - Low，S1 = 2×Pivot - High
- R2/S2 同理

# Step 3 · AI 综合判断（核心逻辑）
**做多信号（Long）**：
- 当前价格进入看涨 Order Block 或 S1 附近（距离 ≤ 2.0%）
- 最近 **2 根** 15 分钟 K 线均为上涨 K 线（收盘 > 开盘）

**做空信号（Short）**：
- 当前价格进入看跌 Order Block 或 R1 附近（距离 ≤ 2.0%）
- 最近 **2 根** 15 分钟 K 线均为下跌 K 线（收盘 < 开盘）

无清晰信号 → 本轮跳过

# Step 4 · 执行下单（市价全仓 4x）
使用 swap_place_order：
- instId = "BTC-USDT-SWAP"
- side = <buy 或 sell>
- ordType = "market"
- posSide = <long 或 short>
- tdMode = "cross"
- sz = <计算使名义价值 ≈ 账户可用余额 × 4 的张数>
- tag = "agentTradeKit"

# Step 5 · 立即设置止盈止损
开仓后立刻调用 swap_place_algo_order：
- 止损：开仓价 × 0.99（多）或 × 1.01（空）
- 止盈：对侧最近的 Order Block 或 R2/S2
- 数量 = 全部持仓

# 严格风控
- 全仓 4x 杠杆
- 单笔止损固定 1%
- 当日浮亏 > 8% 停止开仓
- 仅模拟模式 + 只做 BTC

# 输出要求
每次必须先输出：
1. 当前 BTC 的 Order Block 区间（带 ±1.8% 缓冲，使用1小时K线） + Pivot Points
2. 最近 2 根 15 分钟 K 线判断
3. 完整推理 + 决策
4. 工具调用详情
