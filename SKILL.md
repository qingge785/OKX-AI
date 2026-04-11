---
name: BTC 4x 全仓 OKX撑压线 + 15分钟形态策略 V1.4
description: 使用OKX 1小时自带撑压线指标 + 15分钟K线特定形态入场（第1根触及不破1.5% + 第2根反向K线 → 第3根立即市价全仓4x），模拟盘测试专用。
version: 1.4
author: qingge785
---

# 模拟盘 · 4x全仓 BTC OKX撑压线 + 15分钟形态策略 V1.4（Demo模式）

# 执行节奏
每 15 分钟自动触发一次

# Step 0 · 准备工作
1. 调用 swap_set_leverage 把 BTC-USDT-SWAP 杠杆设为 4，模式 cross
2. 调用 swap_get_positions 检查是否有 BTC 持仓，若有则跳过本轮
3. 调用 account_balance 查询可用余额

# Step 1 · 行情数据采集
- 用 market_get_indicator 获取 OKX 自带**撑压线指标**（1小时图）
  - instId = "BTC-USDT-SWAP"
  - indicator = "support_resistance"
  - bar = "1H"
- 用 market_get_candles 获取 BTC-USDT-SWAP 的 **15分钟 K 线最近 30 根**（用于形态判断）

# Step 2 · OKX 1小时撑压线
AI 提取当前最新的：
- 主要支撑线（Support Lines）
- 主要压力线（Resistance Lines）
- 当前价格与各线的距离

# Step 3 · AI 综合判断（核心新形态逻辑）
严格按以下形态判断（只在第3根15分钟K线时决定是否立即入场）：

**做多信号（Long）**：
- 当前价格已触及或进入 OKX 1小时**支撑线**附近
- 第1根 15分钟K线：触及支撑线，但**最低价没有跌破支撑线下1.5%**
- 第2根 15分钟K线：为**上涨K线**（收盘价 > 开盘价）
- 第3根 15分钟K线：立即触发 → 市价做多

**做空信号（Short）**：
- 当前价格已触及或进入 OKX 1小时**压力线**附近
- 第1根 15分钟K线：触及压力线，但**最高价没有突破压力线上1.5%**
- 第2根 15分钟K线：为**下跌K线**（收盘价 < 开盘价）
- 第3根 15分钟K线：立即触发 → 市价做空

如果不满足以上完整形态 → 本轮跳过

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
- 止损：开仓价 × 0.99（多）或 × 1.01（空）   # 固定1%
- 止盈：对侧最近的 OKX 1小时压力/支撑线
- 数量 = 全部持仓

# 严格风控
- 全仓 4x 杠杆
- 单笔止损固定 1%
- 当日浮亏 > 8% 停止开仓
- 仅模拟模式 + 只做 BTC

# 输出要求
每次必须先输出：
1. OKX 1小时撑压线当前具体位置 + 当前价格距离
2. 最近 3 根 15 分钟 K 线的详细形态判断（是否满足第1根不破1.5% + 第2根反向）
3. 完整推理 + 最终决策（是否在第3根立即入场）
4. 工具调用详情
