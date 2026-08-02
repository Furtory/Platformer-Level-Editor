# 蹬壁跳施加反作用力 — 实现计划

## Context（背景）

当前蹬壁跳（攀登中按跳跃键起跳）只是普通脱离墙面，水平方向由玩家按键决定，没有"反作用力"概念。用户希望新增一个可选的"蹬壁跳施加反作用力"功能：蹬壁跳起跳后，角色被强制朝远离墙壁的方向水平位移，直到达到跳跃最高高度才解除约束。不同模式/缓加减速状态下行为不同，并支持冲刺模式中途用冲刺打断改变方向。

## 需求规格（已与用户澄清）

| 模式 | 缓加减速 | 起跳远离速度          | 方向锁定    | 可改变方向条件               | 冲刺打断                    | 体力                                                 |
| -- | ---- | --------------- | ------- | --------------------- | ----------------------- | -------------------------------------------------- |
| 奔跑 | 关闭   | RUN\_SPD        | 锁定朝远离方向 | 达最高高度才可改方向            | —                       | 远离期间持续消耗；按Shift立即降到WALK\_SPD并保持，松开Shift也不回升，直到最高高度 |
| 奔跑 | 开启   | RUN\_SPD（跳过缓加速） | **不锁定** | 减速到0（非WALK\_SPD）才可改方向 | —                       | 远离期间持续消耗                                           |
| 冲刺 | —    | WALK\_SPD       | 锁定朝远离方向 | 达最高高度才可改方向            | 可消耗1次空中冲刺段数打断，按玩家按键方向冲刺 | 蹬壁跳本身不消耗，冲刺打断走空中冲刺段数                               |

**触发方式**：攀登中按跳跃键（空格/W）起跳自动施加，无需额外按键。
**开关**：新增独立复选框"蹬壁跳施加反作用力"，默认关闭。
**解除时机**：达到跳跃最高高度（velY 由正转 ≤0，即抛物线顶点）时解除反作用力约束。

## 实现方案

### 1. 新增 UI 复选框

**文件**：[Mover.html](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L853-L861)

在"蹬壁补次数"复选框之后、`<hr>` 之前，新增一个复选框行（与现有风格一致，`autocomplete="off"`）：

```html
<div class="row" style="display:flex;align-items:center;gap:8px;margin:4px 0;">
  <input type="checkbox" id="chkWallClimbJumpRecoil" autocomplete="off" style="accent-color:#bd93f9;"/>
  <label for="chkWallClimbJumpRecoil" style="flex:1;font-size:13px;">蹬壁跳施加反作用力</label>
</div>
```

### 2. 新增状态变量与 DOM 引用

**文件位置**：变量定义在 [Mover.html:4555-4557](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L4555)（isClimbing 附近），DOM 引用在 [Mover.html:968-974](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L968)（chkWallClimbGrant 附近）。

```javascript
// DOM 引用（紧随 chkWallClimbReset 引用之后）
const chkWallClimbJumpRecoil = document.getElementById('chkWallClimbJumpRecoil');
let wallClimbJumpRecoilEnabled = false;
if (chkWallClimbJumpRecoil) chkWallClimbJumpRecoil.checked = false; // 显式同步，覆盖表单还原

// 状态变量（紧随 climbDashDir 之后，Mover.html:4557 附近）
let isWallClimbJumping = false;   // 是否处于蹬壁跳反作用力生效中
let wallClimbJumpDir = 0;         // 蹬壁跳远离方向（-1=左, +1=右）
let wallClimbJumpWalkForced = false; // 缓加减速关闭+奔跑模式：按Shift降到WALK_SPD后锁定为走动速度标志
```

### 3. 触发逻辑（doJump 中 isClimbing 分支）

**文件位置**：[Mover.html:6235-6249](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L6235)（doJump 的 `if (isClimbing)` 分支）

在 `isClimbing = false;` 之前插入反作用力初始化：

```javascript
if (isClimbing) {
  segElapsedList = [];
  // 蹬壁跳反作用力：检测墙壁方向，锁定远离方向
  if (wallClimbJumpRecoilEnabled) {
    const wallDir = detectClimbableWallContact();
    if (wallDir !== 0) {
      isWallClimbJumping = true;
      wallClimbJumpDir = -wallDir;        // 远离方向 = 墙壁反方向
      wallClimbJumpWalkForced = false;    // 重置降速锁定标志
    }
  }
}
```

### 4. 顶点解除（loop 中跳跃物理段）

**文件位置**：[Mover.html:6695-6698](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L6695)（`if (isJumping) { airTimeAccum += dt; ... }` 附近）

在更新 maxHeightCur 之后增加顶点检测解除：

```javascript
if (isJumping) {
  airTimeAccum += dt;
  if (posY > maxHeightCur) maxHeightCur = posY;
  // 蹬壁跳反作用力：达到最高高度（velY<=0）时解除
  if (isWallClimbJumping && velY <= 0) {
    isWallClimbJumping = false;
    wallClimbJumpDir = 0;
    wallClimbJumpWalkForced = false;
  }
}
```

落地兜底：在 [Mover.html:6807-6816](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L6807) 落地重置段，追加 `isWallClimbJumping = false; wallClimbJumpDir = 0; wallClimbJumpWalkForced = false;`。同样在 R 键重置 [Mover.html:6006-6024](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L6006) 追加。

### 5. 速度选择与方向锁定（核心，loop 中 spd 选择后）

**文件位置**：[Mover.html:6472-6513](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L6472)（`let spd = WALK_SPD;` 速度选择块）之后、跳跃锁定 [Mover.html:6516](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L6516) 之前插入。

```javascript
/* ---------- 蹬壁跳反作用力：速度与方向覆盖 ---------- */
if (isWallClimbJumping) {
  // 强制方向朝远离墙壁
  dir = wallClimbJumpDir;
  moving = true;
  if (!sprintMode) {
    // 奔跑模式
    if (!smoothSpeedEnabled) {
      // 缓加减速关闭：初始 RUN_SPD；按Shift立即降到WALK_SPD并锁定
      if (shift && !wallClimbJumpWalkForced) {
        wallClimbJumpWalkForced = true;  // 首次按Shift触发降速锁定
      }
      spd = wallClimbJumpWalkForced ? WALK_SPD : RUN_SPD;
    } else {
      // 缓加减速开启：跳过缓加速，直接 RUN_SPD（不锁定方向，但 reversalDecel 目标改为0）
      spd = RUN_SPD;
      currentSpd = RUN_SPD;  // 跳过缓加速：直接拉满
    }
  } else {
    // 冲刺模式：以走动速度远离
    spd = WALK_SPD;
  }
}
```

### 6. 缓加减速适配（reversalDecel 目标改为 0）

**文件位置**：[Mover.html:6538-6544](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L6538)（reversalDecel 判定）与 [Mover.html:6564-6568](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L6564)（reversalDecel 减速分支）

修改 reversalDecel 判定条件：蹬壁跳期间不触发 reversalDecel（因为方向已由反作用力锁定或减速到0才允许改，不会出现 dir !== prevMoveDir 的情况，但为保险显式排除）。

修改缓减速目标：在 reversalDecel 减速分支和 targetIsWalk 缓减速分支中，若 `isWallClimbJumping`，减速下限从 `WALK_SPD` 改为 `0`：

```javascript
if (reversalDecel) {
  const decelPx = decelRate * GRID;
  const floor = isWallClimbJumping ? 0 : WALK_SPD;  // 蹬壁跳期间减速到0
  currentSpd = Math.max(currentSpd - decelPx * dt, floor);
  spd = currentSpd;
}
```

同样对 targetIsWalk 缓减速分支应用 `floor` 逻辑。

### 7. 跳跃锁定兼容

**文件位置**：[Mover.html:6516-6532](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L6516)（jumpLockEnabled 块）

蹬壁跳反作用力与跳跃锁定（chkJumpLock）是独立机制。若两者都启用，蹬壁跳反作用力应在跳跃锁定之前应用（反作用力覆盖 dir/spd 后，跳跃锁定再锁定这些值）。现有顺序已是反作用力在前（Step 5 插入点在跳跃锁定之前），无需额外处理。但需注意：蹬壁跳期间 jumpLockEnabled 会锁定反作用力设置的值，符合预期。

### 8. 冲刺打断（冲刺模式）

**文件位置**：[Mover.html:6388-6444](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L6388)（冲刺进入判定）

现有空中冲刺逻辑已消耗 airSprintSegUsed 并设置 sprintLockTimer。需在冲刺启动时 [Mover.html:6419-6424](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L6419)（isClimbing 分支附近）增加：若 `isWallClimbJumping` 且非攀登中冲刺，则解除蹬壁跳反作用力（让冲刺接管方向）：

```javascript
// 冲刺打断蹬壁跳：解除反作用力，冲刺方向由玩家按键决定
if (isWallClimbJumping) {
  isWallClimbJumping = false;
  wallClimbJumpDir = 0;
  wallClimbJumpWalkForced = false;
}
```

注意：此解除发生在冲刺启动成功后（`if (canSprint)` 块内），保证冲刺方向覆盖远离方向。现有 `climbDashDir` 攀登冲刺逻辑保留不变。

### 9. 体力消耗

**文件位置**：[Mover.html:6634-6622](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L6634)（奔跑模式体力块）

修改 `nowRunning` 条件，使蹬壁跳反作用力生效期间持续消耗体力：

```javascript
const nowRunning = (shift && moving && !isCrouching && !staminaDepleted && !runExitDecel && !reversalDecel)
                || isWallClimbJumping;  // 蹬壁跳反作用力期间持续消耗
```

这样奔跑模式下蹬壁跳远离期间（无论 RUN\_SPD 还是 WALK\_SPD）都消耗体力，符合用户"消耗体力直到达到最高高度"的要求。冲刺模式蹬壁跳本身不消耗体力（走空中冲刺段数）。

### 10. 配置导入导出

**导出**：[Mover.html:1321-1322](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L1321) movement 对象中新增字段：

```javascript
wallClimbJumpRecoil: !!(chkWallClimbJumpRecoil && chkWallClimbJumpRecoil.checked),
```

**导入**：[Mover.html:1564-1575](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L1564) 蹬壁补次数导入块之后新增：

```javascript
try {
  const wcjr = config.movement.wallClimbJumpRecoil;
  const rEn = wcjr === undefined ? false : !!wcjr;  // 缺省关闭
  if (chkWallClimbJumpRecoil) chkWallClimbJumpRecoil.checked = rEn;
  wallClimbJumpRecoilEnabled = rEn;
} catch(e) { /* 忽略 */ }
```

**refreshFormula**：[Mover.html:5481-5487](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L5481) 特殊选项读取段追加：

```javascript
wallClimbJumpRecoilEnabled = !!(chkWallClimbJumpRecoil && chkWallClimbJumpRecoil.checked);
```

**change 事件**：[Mover.html:5561-5576](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L5561) 附近追加：

```javascript
if (chkWallClimbJumpRecoil) chkWallClimbJumpRecoil.addEventListener('change', function() {
  wallClimbJumpRecoilEnabled = !!chkWallClimbJumpRecoil.checked;
});
```

**R 键/模式切换重置**：[Mover.html:6016](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L6016) 与 [Mover.html:1011](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L1011) 追加 `isWallClimbJumping = false; wallClimbJumpDir = 0; wallClimbJumpWalkForced = false;`。

## 关键文件与位置汇总

所有修改集中在单文件 [Mover.html](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html)：

* UI 复选框：L853-861 区域

* DOM 引用与初始化：L968-974 区域

* 状态变量定义：L4555-4557 区域

* doJump 触发：L6235-6249

* 速度选择覆盖：L6472-6513 之后

* 缓加减速适配：L6538-6592

* 顶点解除：L6695-6698

* 冲刺打断：L6419-6424

* 体力消耗：L6634-6622

* 落地/R键重置：L6807-6816, L6006-6024

* 配置导出/导入：L1321-1322, L1564-1575

* refreshFormula：L5481-5487

* change 事件：L5561-5576

## 复用的现有函数

* `detectClimbableWallContact()` [Mover.html:4563](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L4563)：检测墙壁方向，复用于触发时确定远离方向

* `doJump()` 的 isClimbing 分支 [Mover.html:6235](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L6235)：蹬壁跳触发点

* 现有空中冲刺段数消耗逻辑 [Mover.html:6400-6406](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L6400)：冲刺打断复用

* 现有配置导入导出模式 [Mover.html:1564-1575](file:///c:/Users/Furtory/Documents/游戏策划案/移动演示/Mover/Mover.html#L1564)：新字段照搬

## 验证方法

1. **语法验证**：用 Node.js 提取 `<script>` 内容执行 `node --check` 确保无语法错误。
2. **UI 验证**：刷新页面，确认"蹬壁跳施加反作用力"复选框默认未勾选；勾选后可正常切换。
3. **功能验证（奔跑模式 + 缓加减速关闭）**：

   * 绘制可攀登平台，角色跳起贴墙进入攀登，按空格蹬壁跳

   * 观察角色强制朝远离墙壁方向以 RUN\_SPD 移动

   * 蹬壁跳上升途中按住 Shift → 速度立即降到 WALK\_SPD 并保持

   * 松开 Shift → 速度不回升，继续 WALK\_SPD 远离

   * 达到最高高度（顶点）后 → 方向可自由改变

   * 体力条持续下降直到顶点
4. **功能验证（奔跑模式 + 缓加减速开启）**：

   * 蹬壁跳起跳直接 RUN\_SPD（跳过缓加速）

   * 方向不锁定，但反方向需减速到0才可改变

   * 达到顶点解除
5. **功能验证（冲刺模式）**：

   * 蹬壁跳以 WALK\_SPD 远离

   * 途中按 Shift+方向冲刺 → 消耗1次空中冲刺段数，方向变为冲刺方向

   * 不按冲刺则到达顶点后可改方向
6. **开关关闭验证**：复选框未勾选时，蹬壁跳行为与原版一致（无反作用力）。
7. **配置导入导出验证**：导出配置包含 `wallClimbJumpRecoil` 字段；导入后复选框状态正确恢复。
8. **边界验证**：蹬壁跳后落地 / 按 R 重置 → 反作用力状态正确清除，不影响后续行为。

