---
name: BTC 4x 全仓 Donchian 通道策略 V3.0
description: 使用唐奇安通道作为动态支撑压力位 + 15分钟形态入场，有持仓时严格跳过，模拟盘专用
version: 3.0
author: qingge785
---

# 模拟盘 · BTC 4x 全仓 Donchian 通道策略 V3.0

# 执行节奏
每 15 分钟自动触发一次

# Step 0 · 准备工作（优先级最高）
1. 调用 market_get_ticker 获取 BTC-USDT-SWAP 最新实时价格
2. 调用 swap_set_leverage 设置杠杆为 4，模式 cross
3. 调用 swap_get_positions 检查持仓
   - 如果存在任何持仓（数量 > 0），则**立即跳过本轮**，输出“存在持仓，本轮禁止开新单”
4. 调用 account_balance 查询可用余额

# Step 1 · 数据采集
- 调用 market_get_candles 获取 **1小时 K 线最近 100 根**（用于 Donchian 通道）
- 调用 market_get_candles 获取 **15分钟 K 线最近 50 根**（用于形态判断）

# Step 2 · Donchian 通道计算（动态支撑压力）
调用 market_get_indicator 获取唐奇安通道：
- instId = "BTC-USDT-SWAP"
- indicator = "donchian"
- bar = "1H"
- limit = 100
- params = "20"     # 20 周期唐奇安通道（可调整）

AI 提取：
- 上轨（Upper Band）→ 动态压力位
- 下轨（Lower Band）→ 动态支撑位
- 当前价格与上下轨的距离

# Step 3 · 形态判断（仅在无持仓时）
**做多信号（Long）**：
- 当前价格接近或触及 Donchian 下轨（距离 ≤ 0.5%）
- 第1根15分钟K线触及下轨，但最低价未跌破下轨下方1.5%
- 第2根15分钟K线为上涨K线（收盘 > 开盘）
- 第3根15分钟K线触发 → 市价做多

**做空信号（Short）**：
- 当前价格接近或触及 Donchian 上轨（距离 ≤ 0.5%）
- 第1根15分钟K线触及上轨，但最高价未突破上轨上方1.5%
- 第2根15分钟K线为下跌K线（收盘 < 开盘）
- 第3根15分钟K线触发 → 市价做空

如果不满足完整形态或已有持仓 → 本轮跳过

# Step 4 · 执行下单（严格带 posSide）
只有无持仓且信号触发时：
- instId = "BTC-USDT-SWAP"
- side = <buy 或 sell>
- ordType = "market"
- posSide = <long 或 short>     # 必须传递
- tdMode = "cross"
- sz = <根据可用余额计算的4x全仓张数>
- tag = "agentTradeKit"

# Step 5 · 止盈止损
- 止损：多头 = 开仓价 × 0.99，空头 = 开仓价 × 1.01
- 止盈：对侧 Donchian 通道（上轨或下轨）
- 数量 = 全部持仓

# 风控规则
- 有持仓时严格禁止开新单
- 单笔止损固定 1%
- 全仓 4x 杠杆
- 仅模拟盘执行

# 输出要求
每次必须先输出：
1. 当前真实 BTC 价格
2. 当前持仓情况（是否有持仓）
3. Donchian 通道上轨 / 下轨 + 当前价格距离
4. 最近 3 根 15分钟 K 线形态判断
5. 最终决策及工具调用详情（显示 posSide 是否正确传递）
