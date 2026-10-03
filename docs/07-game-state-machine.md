# 回合状态机

单机本地多人依赖明确 FSM，避免找零未完成就切换玩家。

## 状态一览

```
INIT → SETUP → PASS_DEVICE → ROLL_DICE → MOVE → LAND_ON_TILE
  → [分支]
       BUY_PROPERTY? → PAYMENT → (CHANGE?) → CONFIRM
       PAY_RENT?     → PAYMENT → (CHANGE?) → CONFIRM
       CARD?         → DRAW_CARD → RESOLVE_CARD → (PAYMENT?) → ...
       TAX?          → PAYMENT → ...
       JAIL?         → JAIL_TURN → ...
       FREE_PARKING? → END_TURN
  → END_TURN → PASS_DEVICE → (下一玩家) ROLL_DICE
```

终端状态：`GAME_OVER`

## 状态说明

| 状态 | 说明 |
|------|------|
| `INIT` / `SETUP` | 玩家数、名字、颜色、初始 $1500 纸币分配（可自动换面额） |
| `PASS_DEVICE` | 全屏遮罩，等待「我准备好了」 |
| `ROLL_DICE` | 等待掷骰；处理 Double 计数 |
| `MOVE` | 动画移动棋子 |
| `LAND_ON_TILE` | 根据格类型派发到子流程 |
| `PAYMENT` | 付款方 Wallet Panel 凑钞 |
| `CHANGE` | 收款方组合找零（差额 > 0 时） |
| `CONFIRM` | 原子提交钱包与地块变更 |
| `DRAW_CARD` / `RESOLVE_CARD` | 机会/命运 |
| `JAIL_TURN` | 监狱内操作 |
| `MORTGAGE_OR_LIQUIDATE` | 现金不足强制变卖 |
| `BANKRUPTCY` | 破产结算 |
| `END_TURN` | 清理临时 UI 状态 |
| `GAME_OVER` | 胜利展示 |

## 阻塞规则（关键）

1. 处于 `PAYMENT` 或 `CHANGE` 时，**禁止** `END_TURN` 与 `PASS_DEVICE`。
2. `CONFIRM` 成功后才离开结算链。
3. `MORTGAGE_OR_LIQUIDATE` 子状态未完成，禁止回到 `PAYMENT` 之外的无关操作。
4. 练习模式下，`CONFIRM` 前校验失败则留在当前状态。

## 事件（Events）示例

```text
DICE_ROLLED
MOVE_COMPLETE
 TILE_RESOLVED
 PAYMENT_SUBMITTED   // selectedTotal >= due
 CHANGE_SUBMITTED    // changeTotal === requiredChange
 TRANSACTION_CONFIRMED
 TURN_END_REQUESTED  // 仅当非阻塞状态
 DEVICE_HANDOFF_ACK
 PLAYER_BANKRUPT
 GAME_WON
```

## 与存档

持久化字段应包含：

- `fsm.state`
- `fsm.context`（如 `pendingPayment: { payer, payee, due, selected, phase: 'change' }`）

恢复对局时，若停在 `PAYMENT`/`CHANGE`，应直接打开对应 UI。

## 伪代码：转移守卫

```text
on TURN_END_REQUESTED:
  if state in (PAYMENT, CHANGE, MORTGAGE_OR_LIQUIDATE):
    reject
  else:
    transition END_TURN → PASS_DEVICE
```

实现阶段建议使用显式表（from, event, guard, to）或 XState 等库生成可视化图。
