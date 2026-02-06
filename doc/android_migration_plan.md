# Android 本地化迁移方案（Planner → Screen Analyzer → Action Executor）

本文档面向将 `minitap-ai/mobile-use` 的核心链路迁移到 **可直接运行在 Android 手机上的本地 App**。目标是复刻「观察 → 思考 → 执行 → 验证」的自循环，并完成 **感知层（Accessibility + Screenshot）/推理层（OpenAI API）/执行层（Accessibility Action）** 的替换与本地化。  

---

## 1) 环境迁移建议（Kotlin vs Kivy/Python-for-Android）

**推荐：原生 Kotlin + Android Jetpack**
- 与 `AccessibilityService`、`MediaProjection`、`dispatchGesture()` 的系统 API 集成更自然、稳定、可控。
- 性能和稳定性更适合长时间运行的「智能助手」循环。
- 权限、前台服务、通知、无障碍引导等 Android 体验完善。

**可选：Kivy/Python-for-Android**
- 适用于快速验证或已有 Python AI/Agent 逻辑复用场景。
- 对 `AccessibilityService` 和系统权限的交互绕路较多，维护成本高。
- 复杂性和合规风险更高（尤其是无障碍、截图与前台服务策略）。

**结论**：建议使用 **Kotlin 原生** 实现核心链路，必要时通过 gRPC/HTTP 接口与 Python 服务协作，但核心感知与执行应放在 Android 侧。

---

## 2) 目标架构映射：Planner → Screen Analyzer → Action Executor

| 原逻辑模块 | Android 侧替代组件 | 说明 |
| --- | --- | --- |
| Planner | `TaskPlanner`（本地任务分解器，调用 LLM 或本地规则） | 负责生成子目标 |
| Screen Analyzer | `UiAnalyzer` + `ScreenshotProvider` | 通过 AccessibilityService 获取 UI 节点树 + 屏幕截图 |
| Action Executor | `ActionExecutor`（AccessibilityService） | 通过 `performAction()` / `dispatchGesture()` 执行动作 |

**循环机制保留**：  
`Observe` → `Think` → `Act` → `Verify` → `Loop`  
每一轮基于 UI 节点树 + 截图形成可验证的状态快照。

---

## 3) 感知层（Eye）：UI 节点树 + Screenshot

### 3.1 UI 节点树获取

使用 `AccessibilityService#getRootInActiveWindow()` 获取当前窗口的根节点，并遍历生成轻量结构。  
**只保留关键字段**（便于发送给 LLM）：

```json
{
  "id": "com.app:id/title",
  "text": "搜索",
  "desc": "Search button",
  "bounds": "[90,120][240,180]"
}
```

### 3.2 屏幕截图获取

建议两种方案并存：
1. **API 30+** 使用 `AccessibilityService#takeScreenshot()`  
   - 更稳定、无需额外投屏权限（依赖系统支持）。
2. **MediaProjection** 作为兼容方案  
   - 覆盖更广，但需要用户授权且需前台服务。

截图建议压缩成 JPEG/PNG，并可按需缩放到模型输入尺寸（例如 1024px 宽度）。

---

## 4) 执行层（Hand）：无障碍动作执行

核心操作由 `AccessibilityService` 完成：

### 4.1 点击
```kotlin
node.performAction(AccessibilityNodeInfo.ACTION_CLICK)
```

### 4.2 手势（滑动/长按/拖拽）
```kotlin
dispatchGesture(gestureDescription, callback, null)
```

可统一封装成 `ActionExecutor`，支持：
- Tap
- LongPress
- Swipe
- InputText（通过 `ACTION_SET_TEXT`）
- Back/Home 等系统操作

---

## 5) 推理层（Brain）：OpenAI API 接入

### 5.1 输入构造（UI Tree + Screenshot）

将 UI 节点树进行结构化压缩，降低 token 消耗：
- 仅保留 `id/text/desc/bounds`
- 合并重复项
- 过滤无效节点（不可见、无文本、无描述）

发送给 GPT-4o 的 payload 结构建议：

```json
{
  "screen": {
    "ui_nodes": [ ... ],
    "screenshot": "<base64 or image_url>"
  },
  "task": "用户目标",
  "last_action": "...",
  "history": [ ... ]
}
```

### 5.2 输出规范
LLM 返回的动作应结构化：

```json
{
  "action": "tap",
  "target": {
    "text": "搜索",
    "bounds": "[90,120][240,180]"
  },
  "reason": "打开搜索页"
}
```

---

## 6) 自循环机制（Observe → Think → Act → Verify）

### 6.1 Loop Pipeline
1. **Observe**：获取 UI tree + screenshot
2. **Think**：调用 LLM 生成动作或子目标
3. **Act**：执行动作
4. **Verify**：重新获取 UI tree + screenshot，确认变化
5. **Loop**：若目标未完成，继续下一轮

### 6.2 伪代码

```kotlin
while (!goalReached && loopCount < MAX_LOOP) {
  state = observer.capture()
  decision = planner.think(state)
  executor.perform(decision)
  verifier.check(state, observer.capture())
}
```

---

## 7) Android 权限与系统约束

必须处理的核心权限：
- **无障碍服务**：获取 UI tree + 执行动作
- **前台服务**：持续运行 loop
- **截图权限**：MediaProjection（如需）

**注意事项**：
- 无障碍服务需要在系统设置手动开启。
- 截图权限需要前台通知与用户授权。

---

## 8) 组件设计建议

```
App/
 ├─ AccessibilityService
 │   ├─ UiTreeCollector
 │   ├─ ScreenshotProvider
 │   └─ ActionExecutor
 ├─ AgentLoopManager
 │   ├─ TaskPlanner
 │   ├─ DecisionEngine (LLM)
 │   └─ Verifier
 └─ Data
     ├─ UiNodeModel
     └─ ActionModel
```

---

## 9) 最小可行实现（MVP）建议

1. **基础无障碍服务**：获取 UI tree + 点击  
2. **MediaProjection 截图**  
3. **OpenAI API 调用**（只返回 “tap xx”）  
4. **Loop 机制**（最多 5 次）  
5. **本地日志 + UI 调试界面**  

---

## 10) 后续优化方向

- 引入多模态解析：图像辅助定位按钮（视觉对齐）
- 目标检测 + OCR 提升控件识别精度
- Action Replay：失败后回滚策略
- 多任务并行规划（Planner 分拆子目标）

---

## 总结

本迁移方案将 `mobile-use` 的核心「Planner → Screen Analyzer → Action Executor」逻辑，平移到 Android 的 **AccessibilityService + OpenAI API** 体系中，保留了 Loop 机制，同时满足「本地化执行 + 云端推理」的架构目标。
