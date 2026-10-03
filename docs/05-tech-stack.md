# 技术选型与架构

## 约束

- **无后端**：单设备本地多人，全部状态在前端。
- **H5 优先**：浏览器可玩，后续可包 WebApp / 小程序（引擎选型需考虑）。

## 前端引擎（择一）

| 方案 | 适用场景 |
|------|----------|
| **PixiJS / Phaser** | 2D 棋盘、棋子 tween、卡牌与纸币拖拽动画，性能友好 |
| **Cocos Creator / LayaAir** | 编辑器驱动、多平台打包 |
| **Vue 3 / React + Tailwind** | 扁平 UI、DOM 钱包 + Canvas 棋盘混合 |

**文档阶段推荐倾向**：Vue 3 或 React 负责 Wallet / Pass & Play 遮罩 / 表单化规则 UI；PixiJS 或纯 Canvas 负责棋盘与棋子（Phase 1 甚至可用 DOM 网格验证玩法）。

## 状态管理

- 单一 **Game Store**（如 Zustand / Pinia / Redux Toolkit）。
- **有限状态机（FSM）** 驱动回合：禁止在 `PAYMENT_PENDING` 或 `CHANGE_PENDING` 时进入 `END_TURN`。
- 详见 [回合状态机](07-game-state-machine.md)。

### 建议模块划分

```
game/
  fsm/           # 状态机定义与转移
  board/         # 40 格拓扑、坐标
  rules/         # 租金、建房、卡面效果（纯函数）
  wallet/        # 面额运算、凑钞/找零校验
  cards/         # 机会/命运池
  persistence/   # 存档序列化
ui/
  board/         # 棋盘视图
  wallet/        # 钱箱与找零
  pass-and-play/ # 交接遮罩
```

## 本地持久化

- **localStorage**：轻量 checkpoint（整局 JSON，注意大小限制）。
- **IndexedDB**：推荐用于完整历史步进、多存档槽。
- 策略：每次**成功结算**或**回合结束**写入；启动时检测未完成局并提供「继续游戏」。

### 存档最小字段

- `gameId`, `version`, `players[]`, `board`, `fsmState`, `currentPlayerIndex`, `deckStates`, `housesRemaining`, `settings`

## 测试策略（实现阶段）

- `rules/` 与 `wallet/`：**单元测试**（Vitest/Jest）— 租金计算、找零校验、破产判定。
- FSM：**表驱动测试** 非法转移应被拒绝。
- E2E（可选）：Playwright 跑一局极简 2 人剧本。

## 构建与交付（实现阶段）

- Vite + TypeScript
- PWA：`manifest` + Service Worker 缓存静态资源（离线可玩）
- CI：lint + test + 构建 artifact

## 性能目标

- 首屏可交互 < 3s（4G 中端机）
- 拖拽纸币 60fps（避免整板重绘）

## 安全与合规

- 无用户账号则无 PII；存档仅本地。
- 品牌与版权：发行前需独立授权；代码库避免使用 Hasbro 商标资源。
