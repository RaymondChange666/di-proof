# Hummingbot OKX 连接器审计（2026-08-23）

> 范围：hummingbot/connector/exchange/okx/（spot，8 文件 ≈50KB）+ hummingbot/connector/derivative/okx_perpetual/（永续，8 文件 ≈101KB），master @ 2026-08-23
> 方法：我方 OKX 实盘经验驱动的 P0 sweep（订单状态一致性、异常处理、精度/取整、仓位模式）+ 逐行代码核实
> 维护状态：spot 最后一次实质 commit 2026-01-10，永续更早——低维护活跃度，独立审计价值高

## F1 [高] 订单"不存在"检测在状态更新路径被禁用（spot + 永续同款）

- 位置：okx_exchange.py L110-116 · okx_perpetual_derivative.py L139-146
- 事实：`_is_order_not_found_during_status_update_error` 恒返回 False，TODO 注明 "implement this method correctly for the connector"，并点名要修的对应单元测试
- 后果：交易所侧订单消失（风控拒单、系统撤单、过期）时，状态更新轮询无法识别 → 本地 in-flight 订单簿保留幽灵订单，本地状态与交易所真相永久分歧
- 我方经验映射：生产 INCIDENT 011/012 同类（本地状态 vs 交易所真相）；OKX 订单查询的 51603 "Order does not exist" 即应映射的码——永续撤单路径已在正确使用它（见 F2）

## F2 [中] 撤单路径 not-found 检测同样被禁用，但有手工兜底

- `_is_order_not_found_during_cancelation_error` 恒 False（同款 TODO）
- 兜底存在：spot `_place_cancel` 把 51400/51401 当成功；永续把 51603/51401 当成功（constants L186-187 与 OKX 官方错误码一致）
- 残余风险：基类丢失订单跟踪机制整条被关；cancel 兜底只覆盖"撤单那一刻发现"的场景，状态更新轮询路径零兜底

## F3 [低] spot 撤单 instId 直接用 HB 格式 trading_pair

- okx_exchange.py `_place_cancel`：`{"clOrdId":..., "instId": tracked_order.trading_pair}`
- 目前 OKX spot instId 恰好同为 BASE-QUOTE 格式，属格式巧合正确；无 ordId 回退、无显式转换。分类：潜在脆弱性（P2）而非现役 bug

## F4 [低] 永续下单 tdMode 硬编码 "cross"

- okx_perpetual_derivative.py `_place_order` L268：`tdMode="cross"`，无账户仓位模式探测
- 后果：隔离模式账户会被 OKX 拒单；无 mgnMode/tdMode 自动适配。我方实盘经验：仓位模式不匹配是 OKX 高频真实拒单原因之一

## F5 [#8296 相关，假设已标注] WS 账户更新错误循环

- 结构事实：`_listen_for_user_stream_on_url` 捕获一切异常 5 秒后重连 → 任何账户消息处理异常都会形成无限循环（周期 ≈ 连接+认证+订阅+首条账户推送触发崩溃，与报告 ~35 秒吻合）
- 订阅事实：永续订阅 positions/orders（SWAP）+account 三通道；account 未带 ccy（OKX 允许，合法）；posSide 在 net 模式不应发送（与代码一致）
- 假设（未验证，已以条件句写入 #8296 评论区）：崩溃点在账户消息处理而非连接层——依据：报告日志显示认证与订阅成功、REST 正常；且与 #8248（成交后仓位小数崩溃，int() 调用位置类 bug，2026-05-21 报，至今 open）错误签名完全一致
- 处置：定位路径已给（记录原始账户事件 + 与 #8248 解析路径对照）

## 结论

OKX 连接器认证、订阅、基本下单实现正确；订单状态一致性方向存在两个被 TODO 桩禁用的检测路径（F1/F2），属"真实交易所面前会出错的"类别问题。F1 是我方在生产中踩过的同类坑，值得 Hummingbot 侧修复。审计全文同步挂 di-proof（本文件）。

## 出处核对

- spot: https://github.com/hummingbot/hummingbot/blob/master/hummingbot/connector/exchange/okx/okx_exchange.py
- 永续: https://github.com/hummingbot/hummingbot/blob/master/hummingbot/connector/derivative/okx_perpetual/okx_perpetual_derivative.py
- 相关 issue: #8296（WS 错误循环，0 评论）· #8248（仓位小数崩溃，修复中）
