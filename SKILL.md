---
name: BTC 短线 SuperTrend + EMA 策略 V3.0
description: 专为短线设计的 SuperTrend + EMA 组合策略，15分钟到1小时级别，适合杠杆短线交易
version: 3.0
author: qingge785
---

# 模拟盘 · BTC 短线 SuperTrend + EMA 策略 V3.0

# 执行节奏
每 15 分钟自动触发一次（适合短线）

# Step 0 · 准备工作
1. 调用 market_get_ticker 获取 BTC-USDT-SWAP 最新实时价格
2. 调用 swap_set_leverage 设置杠杆为 4，模式 cross
3. 调用 swap_get_positions 检查持仓
   - 如果存在任何持仓，则**立即跳过本轮**，输出“存在持仓，本轮禁止开新单”
4. 调用 account_balance 查询可用余额

# Step 1 · 数据采集
- 调用 market_get_candles 获取 **15分钟 K 线最近 100 根**（主框架）
- 调用 market_get_candles 获取 **1小时 K 线最近 50 根**（EMA 大趋势确认）

# Step 2 · 指标计算
- 调用 market_get_indicator 获取 SuperTrend：
  - indicator = "supertrend"
  - bar = "15m"
  - params = "10,3"        # 常用参数（周期10，乘数3）
- 调用 market_get_indicator 获取 EMA：
  - indicator = "ema"
  - bar = "15m"
  - params = "20"          # EMA20

# Step 3 · 短线判断逻辑
**做多信号（Long）**：
- SuperTrend 线在价格下方（绿色，多头趋势）
- 当前价格位于 EMA20 上方
- 最近 2 根 15分钟K线出现反弹（至少1根阳线）
- 无持仓 → 执行做多

**做空信号（Short）**：
- SuperTrend 线在价格上方（红色，空头趋势）
- 当前价格位于 EMA20 下方
- 最近 2 根 15分钟K线出现回落（至少1根阴线）
- 无持仓 → 执行做空

# Step 4 · 执行下单
只有无持仓且信号触发时：
- instId = "BTC-USDT-SWAP"
- side = <buy 或 sell>
- ordType = "market"
- posSide = <long 或 short>
- tdMode = "cross"
- sz = <根据可用余额计算的合理张数（建议4x但不超过账户30%风险）>
- tag = "agentTradeKit"

# Step 5 · 止盈止损（短线风格）
- 止损：SuperTrend 线反转位置（或固定开仓价 ±1.2%）
- 止盈：1:2 或 1:3 风险收益比（或 SuperTrend 反转）
- 数量 = 全部持仓（或固定比例）

# 风控
- 有持仓时严格禁止开新单
- 单笔风险控制在账户 2-3%
- 仅模拟盘执行

# 输出要求
必须先输出：
1. 当前真实 BTC 价格
2. 当前持仓情况
3. SuperTrend 状态（多头/空头） + EMA20 位置
4. 最终决策及工具调用详情
