---
name: BTC 短线布林带策略 V4.0
description: 短线专用布林带策略，使用布林带上下轨作为动态支撑压力，15分钟级别判断，模拟盘专用
version: 4.0
author: qingge785
---

# 模拟盘 · BTC 短线布林带策略 V4.0

# 执行节奏
每 15 分钟自动触发一次

# Step 0 · 准备工作
1. 调用 market_get_ticker 获取 BTC-USDT-SWAP 最新实时价格
2. 调用 swap_set_leverage 设置杠杆为 4，模式 cross
3. 调用 swap_get_positions 检查持仓
   - 如果存在任何持仓（数量 > 0），则立即跳过本轮，输出“存在持仓，本轮禁止开新单”
4. 调用 account_balance 查询可用余额

# Step 1 · 数据采集
- 调用 market_get_candles 获取 **15分钟 K 线最近 100 根**（主框架）
- 调用 market_get_candles 获取 **1小时 K 线最近 50 根**（大趋势辅助）

# Step 2 · 布林带计算
调用 market_get_indicator 获取布林带：
- instId = "BTC-USDT-SWAP"
- indicator = "bb"          # 或 "boll"
- bar = "15m"
- params = "20,2"           # 周期20，标准差2（经典参数）

提取：
- 上轨（Upper Band）→ 动态压力位
- 中轨（Middle Band）→ 趋势参考
- 下轨（Lower Band）→ 动态支撑位

# Step 3 · 短线入场逻辑
**做多信号（Long）**：
- 当前价格接近或触及布林带下轨（距离 ≤ 0.5%）
- 最近 2 根 15分钟K线至少有1根阳线或明显反弹
- 价格位于中轨下方但开始向上（短期反弹迹象）
- 无持仓 → 执行做多

**做空信号（Short）**：
- 当前价格接近或触及布林带上轨（距离 ≤ 0.5%）
- 最近 2 根 15分钟K线至少有1根阴线或明显回落
- 价格位于中轨上方但开始向下
- 无持仓 → 执行做空

否则跳过

# Step 4 · 执行下单
只有无持仓且信号触发时：
- instId = "BTC-USDT-SWAP"
- side = <buy 或 sell>
- ordType = "market"
- posSide = <long 或 short>
- tdMode = "cross"
- sz = <根据可用余额计算的合理张数（建议单笔风险控制在账户2-3%）>
- tag = "agentTradeKit"

# Step 5 · 止盈止损（短线风格）
- 止损：固定开仓价 ±1%
- 止盈：1:2.5 风险收益比（或价格触及对侧布林带轨）
- 数量 = 全部持仓（或固定比例）

# 风控规则
- 有持仓时严格禁止开新单
- 单笔风险控制在账户 2-3%
- 仅模拟盘执行

# 输出要求
必须首先输出：
1. 当前真实 BTC 价格
2. 当前持仓情况
3. 布林带上轨 / 中轨 / 下轨数值 + 当前价格距离
4. 15分钟K线形态判断
5. 最终决策及工具调用详情
