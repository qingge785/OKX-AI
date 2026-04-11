# 模拟盘 · 4x全仓 BTC Order Block + Pivot Points 策略 V1.0（Demo模式）

# 执行节奏
每 15 分钟自动触发一次

# Step 0 · 准备工作（每次触发都先执行）
1. 调用 swap_set_leverage 把 BTC-USDT-SWAP 杠杆设为 4，模式 cross
2. 调用 swap_get_positions 检查是否有 BTC 持仓，若有则跳过本轮
3. 调用 account_balance 查询可用余额

# Step 1 · 行情数据采集（仅 BTC）
- 用 market_get_candles 获取 BTC-USDT-SWAP 的 **4小时 K 线最近 100 根**（用于识别 Order Block 和 Pivot Points）
- 用 market_get_candles 获取 BTC-USDT-SWAP 的 **15分钟 K 线最近 30 根**（用于 3 连阳/阴判断）

# Step 2 · 支撑压力位识别（Order Block + Pivot Points）
AI 请严格按以下规则识别：

**Order Block（主要信号）**：
- 看涨 Order Block（多头支撑）：最近一次强上涨前的最后一根**阴线**（实体收在低点附近），作为潜在买入区。
- 看跌 Order Block（空头压力）：最近一次强下跌前的最后一根**阳线**（实体收在高点附近），作为潜在卖出区。
- 只取**最近 1-2 个未被完全突破的 Order Block**，并标注是否被测试过。

**Pivot Points（辅助确认）**：
用最近一根 4h K 线计算标准 Pivot：
- Pivot = (High + Low + Close) / 3
- R1 = 2×Pivot - Low，S1 = 2×Pivot - High
- R2/S2 同理
当前价格接近 S1/R1 或 Order Block 时优先级更高。

# Step 3 · AI 综合判断（核心逻辑）
**做多信号（Long）**：
- 当前价格触及或进入看涨 Order Block 或 S1 附近（距离 ≤ 0.5%）
- 最近 2 根 15 分钟 K 线均为上涨 K 线（收盘 > 开盘）

**做空信号（Short）**：
- 当前价格触及或进入看跌 Order Block 或 R1 附近（距离 ≤ 0.5%）
- 最近 2 根 15 分钟 K 线均为下跌 K 线（收盘 < 开盘）

无信号 → 跳过

# Step 4 · 执行下单（仅 BTC 市价全仓 4x）
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
- 单笔止损固定 1%（对应账户约 4% 风险）
- 当日浮亏 > 8% 停止开仓
- 仅模拟模式 + 只做 BTC

# 输出要求
每次必须先输出：
1. 当前 BTC 的 Order Block 位置 + Pivot Points（S1/R1/R2/S2）
2. 15 分钟 3 根 K 线判断
3. 完整推理 + 决策
4. 工具调用详情
