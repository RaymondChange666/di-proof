# KILL SWITCH 激活记录（2026-08-23）

## 状态：hrscan 新开仓已暂停

- 机制：/root/bot/KILL_SWITCH 存在 → 每 tick 仓位管理（trail_stops/OCO）照常，新开仓完全阻断（日志每 tick 打印 "KILL SWITCH ACTIVE — no new positions"）
- 激活时间：2026-08-23（T2 对抗审查通过 + 总裁部署令）
- 实盘状态：空仓（激活时 OKX 无持仓，零仓位层面副作用）

## 原因（如实记录）

31 个月严格回测证实生产配置（V0 含 tf_confirm）信号层负期望：235 笔、胜率 16%、收益 -0.8%。五轮独立标准验证全部证伪：
hrscan 信号层 / 策略 D / Alpha Factory 已测因子 / Alpha Factory 未测因子 / Supertrend+Breakout20。
低频率只是掩盖了负斜率，不是过滤器创造了正期望。

## 解除的唯一正当依据

有新的、通过同样严格标准（next-bar-open、真实费率+滑点、train/OOS 分离、样本量门槛 ≥30）验证过的正期望信号，经 T3（总裁裁决）+ T2（对抗审查）流程。
不是"冻结期到了"，不是"感觉可以试试"。

## 过程发现（同样值得记录）

KILL_SWITCH 机制此前只存在于技能文档（di-production-ops），从未写进过 v5 代码（grep 0 命中、全目录 0 命中、git log -S 空）。文档与代码漂移通过一次跨系统指令传递才被发现。教训：文档说存在 ≠ 代码存在，任何"现成机制"执行前先三重核实。
