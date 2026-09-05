# 当前研究主线：Scene-Conditioned Sparse Whole-Body Trajectory Planning

> **治理规则：** 本文件保存当前唯一有效研究主线。未经用户明确要求，Codex 不得修改本文件。下述主线内容继承自用户于 2026-09-03 确认的版本。

更新时间：2026-09-03

## 一、最终研究目标

目标是实现 Unitree G1 在复杂三维室内环境中的自主全身无碰撞穿越，包括：

* 跨越低障碍；
* 下蹲通过顶部障碍；
* 侧身通过狭窄通道；
* 绕行；
* 根据障碍几何自适应调整身体姿态。

核心问题不是普通二维导航，也不是单纯生成“看起来合理”的人体动作，而是：

> 根据三维环境约束，为 G1 规划未来若干秒的稀疏全身运动意图，并利用强运动生成模型补全为自然、连续的全身运动，最后通过物理控制策略在机器人上执行。

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
Scene-Conditioned Sparse Trajectory Planner
        【核心研究模块】
            ↓
Future Sparse Spatiotemporal Constraints
            ↓
      ARDY / Kimodo
            ↓
Dense Whole-Body Kinematic Motion
            ↓
       SONIC / GEAR-SONIC
            ↓
       Unitree G1 Control
            ↓
     Physics / Real Robot
```

后续采用 receding-horizon：

```text
感知环境
↓
预测未来约 1~3 s sparse trajectory
↓
ARDY 补全 dense motion
↓
SONIC 执行前一小段
↓
重新感知
↓
重新规划
```

---

## 三、核心研究模块到底输出什么

当前不以直接预测：

```text
29 DoF joint angles
完整 G1 motion
motor action
torque
```

作为主要路线。

上层 Planner 输出低维、可解释的未来时空约束，例如：

```text
Root:
    future x / y / yaw waypoints

Left Foot:
    sparse future xyz positions

Right Foot:
    sparse future xyz positions

Pelvis:
    sparse height / position constraints

Torso:
    sparse position / orientation constraints
```

可以表示成：

K = {K(t1), K(t2), ..., K(tN)}

其中 N 较小，例如 5~20 个关键时间点，而不是生成几十或几百帧完整关节轨迹。

第一版优先研究：

```text
root
+
left/right feet
+
pelvis
+
torso
```

之后根据实验决定是否增加：

```text
head
hands
knees
其他身体部位
```

---

## 四、为什么采用 Sparse Planning + Motion Prior

核心分工：

```text
Planner：
决定“身体关键部位应该经过哪里”

ARDY / Kimodo：
决定“完整身体具体应该怎么自然地运动”

SONIC：
决定“电机如何真正把这个动作执行出来”
```

例如跨越箱子时：

Planner 只需要决定：

```text
左脚何时抬起
左脚经过哪里
左脚何时落地
root 如何前进
pelvis 如何变化
```

不需要自己决定：

```text
膝关节每一帧多少度
踝关节每一帧多少度
手臂如何摆动
所有中间帧如何平滑连接
```

这些自由度交给已经具有强运动先验的 ARDY / Kimodo。

这样把：

规划问题

与：

运动补全问题

以及：

物理控制问题

明确解耦。

---

## 五、核心学习问题

希望训练：

p(K_future | Environment, G1_state, Goal)

即：

```text
环境约束
+
当前机器人状态
+
目标
↓
未来 sparse whole-body trajectory
```

由于同一障碍可能存在：

```text
左脚先跨
右脚先跨
绕行
侧身
不同落脚点
```

因此这是天然的多模态生成问题。

当前优先尝试：

Conditional Flow Matching / Rectified Flow

而不是单一确定性回归。

---

## 六、第一阶段环境输入

不要一开始直接上复杂 Point Cloud。

按照以下顺序验证：

V0：

```text
单一 primitive obstacle 参数
[x, y, z, width, depth, height]
```

V1：

```text
多个 primitive obstacles
```

V2：

```text
occupancy / local voxel / ESDF
```

V3：

```text
partial point cloud
```

V4：

```text
真实深度感知
```

首先验证 Sparse Planner 这个核心假设是否成立，再增加感知复杂度。

---

## 七、第一阶段模型输出

第一版可以固定 N 个 future anchors，例如：

```text
N = 8
```

每个 anchor：

```text
root_xy / yaw
left_foot_xyz
right_foot_xyz
pelvis_z
torso orientation / position
```

或者先进一步简化成：

```text
root
+
left foot
+
right foot
```

确认模型可以根据障碍产生明显不同的 traversal strategy，再逐渐增加身体约束。

---

## 八、下游模型角色

### ARDY

长期目标中优先作为 online motion completion / streaming motion generator。

输入：

```text
history motion
+
future sparse constraints
```

输出：

```text
dense whole-body motion
```

适合最终 receding-horizon 系统。

### Kimodo

当前优先作为：

```text
offline motion completion baseline
constraint feasibility tester
data generator
oracle / teacher
```

非常适合第一阶段检查：

> Planner 预测出的 sparse constraints 能否被强运动先验补全成合理 G1 motion。

### SONIC / GEAR-SONIC

负责：

```text
dense kinematic reference
↓
physics-based tracking
↓
joint control
```

不要求上层 Planner 学习电机控制。

---

## 九、当前主线明确不做什么

以下内容目前不是核心主线：

### 1. 直接训练 Text-to-G1 full-body model

例如直接让 HYMotion-G1 输出完整 29 DoF trajectory。

可以作为 baseline，但不是当前主要方法。

### 2. 直接让 Flow Matching 输出 motor action

规划和控制保持解耦。

### 3. 一开始加入 VLM / VLA

第一阶段先验证几何约束条件下的 sparse planning。

语言指令属于后续扩展。

### 4. 一开始使用复杂真实点云

先用 ground-truth primitive geometry 验证核心模型。

### 5. 自己重新训练完整 motion prior

优先利用 ARDY / Kimodo 已经学到的运动先验。

---

## 十、训练数据定义

一个训练样本应至少包含：

```text
Environment
Current G1 State
Goal
Future Sparse Anchors
```

即：

(condition, target_sparse_trajectory)

可以额外保存完整 dense motion：

```text
dense whole-body trajectory
```

用于：

* 从 dense motion 自动提取 sparse anchors；
* 验证 sparse constraints 的合理性；
* 计算 collision / clearance；
* 后续研究不同 anchor representation。

数据来源可以逐渐包括：

```text
程序化 expert trajectory
trajectory optimization
RL successful rollout
Kimodo / ARDY generated motion
motion tracking rollout
人工设计的简单 constraint templates
```

---

## 十一、模型评价必须包含

不能只评价 trajectory MSE。

重点指标：

```text
Sparse Anchor Error
Goal Success Rate
Whole-body Collision Rate
Minimum Clearance
Motion Completion Success
ARDY / Kimodo Constraint Satisfaction
SONIC Tracking Success
Fall Rate
Traversal Success Rate
Inference Time
```

最终判断标准始终是：

> Planner 给出的稀疏运动意图，经过 motion completion 和 physical tracking 后，G1 是否真的可以安全穿过障碍。

---

## 十二、当前研究创新空间

当前最核心的问题不是：

> Flow Matching 能不能生成 trajectory？

而是：

> 如何利用三维环境约束，生成少量但足以决定人形机器人全身穿越策略的时空 sparse anchors？

可能研究：

1. Sparse whole-body anchor representation；
2. Scene-conditioned Flow Matching；
3. 不同身体部位的 adaptive anchors；
4. obstacle/body clearance-aware training；
5. multi-modal traversal strategy generation；
6. planner 与 ARDY / Kimodo motion prior 的接口设计；
7. receding-horizon sparse motion planning；
8. 根据下游 tracker 可执行性反向优化 Planner。

---

## 十三、当前路线一句话定义

> 面向复杂三维环境，训练一个 Scene-Conditioned Sparse Whole-Body Trajectory Planner，根据环境、机器人当前状态和目标生成未来稀疏身体关键点轨迹，再利用 ARDY/Kimodo 补全全身运动，并通过 SONIC 在 Unitree G1 上物理执行。

除非实验明确证明该分层假设不成立，否则当前不切换到“直接生成完整 G1 joint trajectory”的主路线。
