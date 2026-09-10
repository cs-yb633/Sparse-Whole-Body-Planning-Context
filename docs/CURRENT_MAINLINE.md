# 当前研究主线：Scene-Conditioned Sparse Whole-Body Trajectory Planning

> **治理规则：** 本文件保存当前唯一有效研究主线。未经用户明确要求不得修改。历史 reference repo 的审计结论不等于当前主线；实现底座也不等于研究贡献。

更新时间：2026-09-10

## 一、最终研究目标

目标是实现 Unitree G1 在复杂三维环境中的自主全身无碰撞穿越，包括：

- 跨越低障碍；
- 下蹲通过顶部障碍；
- 侧身通过狭窄通道；
- 绕行；
- 根据障碍几何自适应调整身体姿态。

核心问题不是普通二维导航，也不是直接生成完整人体/G1 动作，而是：

> **根据三维环境、当前 G1 状态和目标，生成未来若干秒少量、具有身体语义和时间语义的全身运动锚点，再交给运动先验补全为完整运动并由物理控制器执行。**

---

## 二、当前默认系统架构

```text
3D Environment
Point Cloud / ESDF / Obstacle Geometry
            +
Current G1 State
            +
Goal
            ↓
Scene / Constraint Encoder
            ↓
Scene-Conditioned Sparse Whole-Body Planner
        【核心研究模块】
            ↓
Timed Semantic Whole-Body Anchors
            ↓
ARDY / Kimodo / other motion prior
            ↓
Dense Whole-Body Kinematic Motion
            ↓
SONIC / GEAR-SONIC / tracker
            ↓
Unitree G1
            ↓
Physics / Real Robot
```

长期采用 receding-horizon：

```text
感知环境
↓
预测未来约 1~3 s sparse anchors
↓
运动补全
↓
执行前一小段
↓
重新感知
↓
重新规划
```

---

## 三、核心学习问题

希望学习：

$$
p(K_{future}\mid E,S_{G1},G)
$$

其中：

- `E`：三维环境/障碍约束；
- `S_G1`：当前机器人状态；
- `G`：目标；
- `K_future`：未来稀疏全身语义锚点。

同一环境可能存在多种合法 traversal strategy，例如：

```text
左脚先跨
右脚先跨
绕行
侧身
不同落脚点
不同躯干姿态
```

因此核心 Planner 是一个多模态条件生成问题。

当前优先采用 **Conditional Flow Matching** 作为生成范式。

Flow Matching 是生成器实现选择，不是论文核心贡献本身。

---

## 四、Timed Semantic Anchors 是核心表示

Planner 不直接预测：

```text
完整 29 DoF joint trajectory
motor action
torque
完整 dense whole-body motion
```

而是预测少量具有明确身体语义与时间语义的 task-space anchors。

长期可包括：

```text
root
left foot
right foot
pelvis
torso
```

必要时再增加：

```text
head
hands
knees
其他身体部位
```

关键定义：

> **Timed Semantic Anchor 表示“在真实未来时间 t，某个身体部位应该位于/朝向哪里”。**

它不是 B-spline control point，也不是纯数学曲线系数。

B-spline 可以未来用于平滑、碰撞采样或轨迹 stitching，但当前不属于核心 representation。

---

## 五、当前 V0 固定表示

第一版只使用：

```text
root
left foot
right foot
```

固定 8 个未来时刻。

每个时刻 9 个值：

```text
root_x
root_y
root_yaw
left_foot_x
left_foot_y
left_foot_z
right_foot_x
right_foot_y
right_foot_z
```

因此：

$$
K\in\mathbb R^{8\times9}
$$

这 8 个位置对应固定的 future timestamps，必须保持真实时间语义。

---

## 六、当前 V0 环境输入

暂时只使用简单向量条件：

```text
single primitive box geometry
+
current root / feet state
+
goal
```

例如：

```text
box center xyz
box size xyz
current root x/y/yaw
current left/right foot xyz
goal x/y/yaw
```

当前不引入：

```text
PointNet
Point Transformer
point cloud
voxel
复杂 ESDF encoder
VLM / VLA
真实深度感知
```

先验证 Planner 是否真的学习：

```text
scene geometry
→
semantic body-part trajectory
```

---

## 七、当前生成模型实现底座

当前选定的实现起点是：

> **HRI-EU/flow_matching 中的 ConditionalUnet1D + vector global_cond/FiLM + independent Conditional Flow Matching。**

选择它的原因是：

- 原生 `[B,T,D]` 轨迹接口可自然改为 `[B,8,9]`；
- 已有连续向量 `global_cond` 条件路径；
- temporal Conv1d 显式建模时间维；
- independent CFM 训练路径简单；
- 已完成隔离核心 `[B,8,9]` smoke test。

但必须明确：

> **HRI Flow Matching 只是 implementation base/backbone，不是当前研究主线，也不是研究贡献。**

未来可以替换 velocity backbone 或 Flow Matching variant，而不改变核心研究问题。

已知 HRI example 中 sampling 初始分布存在 `rand` 与训练 `randn` 不一致的问题；任何正式 fork 必须使用与训练一致的 Gaussian initialization。

MotionFM 保留为 Transformer motion-sequence architecture reference，不作为当前首选实现底座。

---

## 八、历史 reference repo 的当前状态

### FlowMP

已完成用于理解 Flow Matching 数学、trajectory generation、sampling 和 conditioning 的历史审计。

它不再是当前 base model，也不再阻塞实现。

### ABPolicy

已完成用于理解低维 trajectory parameterization 与 receding-horizon refitting 的历史审计。

其 B-spline control points 与本项目 Timed Semantic Anchors 本质不同。

它不再是当前主线参考，不再继续深入。

### SanD-Planner

仅保留为未来 scene encoding、safety evaluation、candidate selection、ESDF 与 replanning 的可选系统参考。

它不是当前 V0 的 blocker，也不需要先完成完整 audit 才能开始 V0。

---

## 九、为什么采用 Sparse Planning + Motion Prior

模块分工保持不变：

```text
Planner：
决定关键身体部位未来应该经过哪里、何时经过

Motion Prior：
决定完整身体如何自然、连续地实现这些约束

Tracker：
决定机器人如何在物理系统中执行 reference motion
```

例如跨越箱子时，Planner 应决定：

```text
哪只脚先跨
脚何时抬起
脚需要抬多高
落脚点在哪里
root 如何前进
未来 pelvis / torso 如何调整
```

不要求 Planner 决定每一帧所有关节角度。

---

## 十、当前阶段的数据策略

V0 使用 synthetic expert data。

一个训练样本至少包含：

```text
Environment condition
Current G1 state
Goal
Target 8×9 Timed Semantic Anchors
```

第一版可以只提供一种 traversal strategy，例如 left-foot-first。

程序化 expert 必须显式满足障碍 clearance，例如：

$$
z_{foot,max} > h_{box}+margin
$$

当前数据真实性不是首要目标；首要目标是建立明确、可验证的 scene-to-anchor 映射。

---

## 十一、当前第一个核心实验 EXP-001

固定：

```text
same current G1 state
same goal
same obstacle position / width / depth
```

只改变 box height，例如：

```text
0.10 m
0.20 m
0.30 m
```

核心验证：

$$
h_{obstacle}\uparrow
\Rightarrow
h_{foot-anchor}\uparrow
$$

必须至少报告：

```text
predicted max left/right foot height
required clearance
anchor error
box height vs predicted max foot height figure
```

并做 condition sanity check，确认模型不是忽略 box geometry。

如果 EXP-001 不成立，先解决 condition→trajectory 学习问题，不提前引入复杂感知、下游 motion prior 或更复杂 solver。

---

## 十二、第二阶段再验证多模态

只有 EXP-001 成功后，再构造：

```text
left-foot-first
right-foot-first
```

等多种 demonstration。

对同一 condition 从不同随机噪声采样，验证 Flow Matching 是否能够生成不同合法 traversal strategy。

该实验用于回答为什么需要 generative planner，而不是单一 deterministic regression。

---

## 十三、环境输入的长期升级顺序

保持逐步增加复杂度：

```text
V0: single primitive obstacle vector
V1: multiple primitive obstacles
V2: occupancy / local voxel / ESDF
V3: partial point cloud
V4: real depth perception
```

只有前一阶段核心假设得到验证，才增加感知复杂度。

---

## 十四、下游角色

### Motion Completion

长期优先考虑 ARDY / Kimodo 或其他强 motion prior：

```text
sparse anchors
+
history motion
↓
dense whole-body motion
```

具体选择仍需后续实验验证，不提前锁死。

### Physical Tracking

SONIC / GEAR-SONIC 或其他 G1 whole-body tracker 负责：

```text
dense reference
↓
physics-based tracking
↓
robot execution
```

当前 V0 不接下游。

---

## 十五、当前明确不做什么

在 EXP-001 成功以前，不主动加入：

- Point Cloud / depth perception；
- VLM / VLA；
- 完整 ESDF 系统；
- ARDY / Kimodo 集成；
- SONIC / 真机；
- 大规模 RL；
- 完整 indoor benchmark；
- B-spline 作为核心 anchor representation；
- 直接生成完整 G1 joint trajectory；
- Flow Matching solver/architecture 的大规模调参。

---

## 十六、研究贡献应围绕什么

当前研究贡献不应表述为：

> “把 Flow Matching 用到 G1 上。”

也不应只是：

> “把某个人体/机械臂模型迁移到 humanoid。”

真正核心的问题是：

> **如何根据三维场景与任务目标，生成少量但足以表达 humanoid whole-body traversal intention 的 Timed Semantic Whole-Body Anchors？**

潜在贡献集中在：

1. Timed Semantic Whole-Body Anchor representation；
2. Scene-conditioned sparse whole-body generative planning；
3. multi-modal traversal strategy generation；
4. obstacle/body clearance-aware planning；
5. sparse planner 与 motion prior / tracker 的分层接口；
6. receding-horizon whole-body sparse planning。

Flow Matching 和 HRI backbone 都是实现这些目标的技术手段。

---

## 十七、最终判断标准

最终系统必须回到：

```text
3D environment
↓
合理的 sparse whole-body intention
↓
可补全的完整动作
↓
物理可跟踪
↓
whole-body collision-free
↓
G1 完成 traversal
```

当前 V0 的成功标准则严格更小：

```text
simple obstacle condition
↓
Conditional Flow Matching
↓
8×9 Timed Semantic Anchors
↓
模型输出对 obstacle geometry 有明确、正确响应
```

---

## 十八、当前路线一句话定义

> **面向复杂三维环境，学习一个 Scene-Conditioned Generative Sparse Whole-Body Planner，根据环境、G1 当前状态和目标生成未来 Timed Semantic Whole-Body Anchors，再利用运动先验补全完整动作并通过物理 tracker 执行；当前 V0 以 HRI ConditionalUnet1D + Conditional Flow Matching 作为实现底座，先验证单 box 条件到 8×9 root/feet anchors 的最小闭环。**
