# 当前阶段任务目标：验证 Sparse Planner 最小闭环

更新时间：2026-09-05

> 本文件是当前执行约束，优先级高于 `ROADMAP.md`。长期方向以 `CURRENT_MAINLINE.md` 为准；当前只实现完成本阶段 milestone 所必需的内容。

## 0. 当前阶段一句话目标

当前阶段只需要回答一个问题：

> **Conditional Flow Matching 能否根据简单障碍条件，生成合理的 G1 稀疏未来身体关键点轨迹（Timed Semantic Anchors）？**

当前阶段不是为了完成最终系统。

不要提前解决：

* 完整 G1 运动生成；
* ARDY / Kimodo 集成；
* SONIC 控制；
* Point Cloud 感知；
* ESDF 完整系统；
* 真机；
* 大规模 RL；
* 完整复杂室内场景。

---

# 1. 当前核心假设

我们希望学习：

$$
p(K_{future}\mid E, S_{G1}, G)
$$

其中：

* \(E\)：环境/障碍条件；
* \(S_{G1}\)：当前 G1 状态；
* \(G\)：目标；
* \(K_{future}\)：未来稀疏身体关键点轨迹。

第一版只验证：

```text
简单 obstacle geometry
+
简单 current G1 state
+
goal
↓
Conditional Flow Matching
↓
Sparse Timed Semantic Anchors
```

---

# 2. 第一版不要做完整 whole-body

V0 输出先固定为：

```text
root
left foot
right foot
```

固定 8 个未来时刻，例如未来约 2 秒：

```text
t1 ... t8
```

每个时间点：

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

即：

$$
K\in\mathbb R^{8\times9}
$$

共 72 个连续变量。

注意：

这些不是 B-spline control points。

它们是：

> **Timed Semantic Anchors**

例如：

> t = 0.75 s 时，左脚应该实际位于某个 xyz。

后续需要能够直接转换成 Kimodo / ARDY 的 sparse constraints。

---

# 3. 当前 V0 环境也必须保持简单

暂时只使用：

```text
单个低障碍 box
```

condition 可以先是：

```text
box center xyz
box size xyz
goal
current root/feet state
```

即简单向量。

不要现在加入：

```text
PointNet
Point Transformer
depth image
point cloud
voxel
复杂 ESDF encoder
VLM
```

原因：

当前需要验证的是 Sparse Planner 本身，而不是感知能力。

---

# 4. 当前三个 reference repo 的角色

已经 clone：

* FlowMP
* ABPolicy
* SanD-Planner

它们只是 reference code。

不要试图完整复现三个项目。

不要把任何一个仓库直接改造成最终项目。

## FlowMP

当前只学习：

```text
Conditional Flow Matching
trajectory representation
training loss
sampling
condition injection
```

目标：

> 能跑通一个最小 3D CFM example，并真正理解 x0 / x1 / xt / velocity target / sampling。

不要为了 FlowMP 的所有 feature、环境和 notebook 做完整复现。

---

## ABPolicy

当前只学习：

```text
Flow Matching
→
低维 sparse control-point representation
```

重点理解：

* `n_ctrl = 8`；
* dense trajectory 如何转换成低维表示；
* Flow 如何直接生成少量 points；
* receding-horizon trajectory continuity 怎么处理。

不要：

* 接 Piper 机械臂；
* 配真实相机；
* 复现完整 manipulation pipeline。

---

## SanD-Planner

当前只学习系统结构：

```text
scene condition
↓
generative planner
↓
sparse trajectory representation
↓
safety evaluation
↓
replanning
```

重点阅读：

* dataset；
* condition encoder；
* scene encoder；
* sparse trajectory representation；
* safety / ESDF evaluator；
* closed-loop structure。

不要：

* 为了 SanD 单独安装旧 Isaac Lab；
* 复现完整 navigation benchmark；
* 大规模下载 dataset/checkpoint；
* 修改当前项目 Isaac 环境。

---

# 5. 当前实际开发顺序

必须按下面顺序进行。

## Phase A — FlowMP 最小复现

完成以下内容即可：

1. 跑通最小 3D trajectory generation；
2. 找到 training loss；
3. 找到 sampling/integration；
4. 明确 trajectory tensor shape；
5. 明确 condition tensor；
6. 明确 obstacle information 是否进入模型以及如何进入；
7. 画一张生成轨迹图。

完成后停止继续深入 FlowMP。

---

## Phase B — ABPolicy 最小代码审计

回答：

1. sparse/control-point target 如何构造；
2. Flow model 输入输出 shape；
3. 8 个 control points 如何生成；
4. training loss；
5. continuity/refitting 如何实现。

不要求完整跑机器人系统。

完成后停止。

---

## Phase C — SanD 最小代码审计

回答：

1. scene information 如何编码；
2. goal 如何 condition；
3. sparse target 是什么；
4. safety evaluation 怎么做；
5. online replanning pipeline 如何组织。

不要求完整训练。

完成后停止。

---

# 6. 然后立即开始自己的 V0

三个 reference repo 的目的只是帮助我们写：

```text
src/sparse_planner/
```

自己的第一版应该非常小：

```text
sparse_planner/
├── dataset.py
├── anchor_representation.py
├── condition_encoder.py
├── flow_model.py
├── train.py
├── sample.py
└── visualize.py
```

不要先建立大型 framework。

---

# 7. 第一批训练数据

第一版使用 synthetic expert data。

例如随机采样：

```text
box height
box position
goal position
```

然后程序化产生合理的：

```text
root anchors
left-foot anchors
right-foot anchors
```

例如跨越时满足：

$$
z_{foot} > h_{box} + margin
$$

先生成约：

```text
1k ~ 10k samples
```

即可。

当前阶段不追求数据真实性。

当前目的只是验证：

> 模型是否真正利用 obstacle condition 学习 sparse trajectory mapping。

---

# 8. 当前必须完成的第一个科研实验

固定：

```text
same current G1 state
same goal
```

改变：

```text
box height
```

例如：

```text
0.10 m
0.20 m
0.30 m
```

期望：

```text
obstacle 越高
↓
模型生成的 foot anchor 越高
```

必须画出：

```text
obstacle height
vs
predicted maximum foot height
```

如果模型完全不随 obstacle 改变：

当前假设没有得到验证，需要先解决这一问题。

---

# 9. 第二个实验：多模态

同一个 obstacle condition 下提供两类 demonstration：

```text
left-foot-first
right-foot-first
```

然后对同一个 condition 使用多个随机 noise sample。

检查 Flow Matching 是否能够得到：

```text
sample A → 左脚先跨
sample B → 右脚先跨
```

该实验用于验证：

> 为什么需要生成模型，而不是普通 deterministic MLP regression。

---

# 10. 当前阶段完成标准

以下全部满足，才允许进入下一阶段：

### Reference understanding

* FlowMP 最小 CFM 跑通；
* ABPolicy sparse generation 机制看懂；
* SanD scene-conditioned planning pipeline 看懂。

### Own model

* 自己的 Timed Semantic Anchor representation 已确定；
* 自己的最小 CFM 能训练；
* synthetic dataset 能生成；
* inference 能生成 8×9 sparse anchors。

### Core experiment

模型能够表现出：

$$
h_{obstacle}\uparrow
\Rightarrow
h_{foot-anchor}\uparrow
$$

并且能够可视化。

### Optional second milestone

同一 obstacle 能从不同噪声产生至少两种不同 traversal strategy。

---

# 11. 当前阶段明确禁止钻牛角尖的问题

如果遇到下面的问题，但不阻塞核心实验，记录下来后继续，不要长期停留：

* 最优 Flow Matching solver 是什么；
* Euler / RK4 哪个最好；
* Transformer 是否一定优于 MLP；
* anchor 数量到底应该是 6、8、10 还是 12；
* quaternion / 6D rotation 最终哪个好；
* PointNet++ 还是 Point Transformer；
* G1 所有 collision geometry 如何精确建模；
* ARDY 与 Kimodo 最终谁更好；
* SONIC 最终部署细节；
* 数据规模到底需要 10k 还是 1M；
* 是否应该提前引入 VLM；
* 是否应该直接改某个最新 foundation model。

当前原则：

> **能用简单版本验证核心假设，就先用简单版本。**

如果一个问题不影响：

```text
condition
→
CFM
→
8×9 sparse anchors
```

跑通，就不是当前 blocker。

---

# 12. Codex 工作方式要求

发现问题时先判断：

```text
这个问题是否阻塞当前最小实验？
```

如果否：

1. 记录到 `OPEN_QUESTIONS.md`；
2. 采用最简单合理默认值；
3. 继续主任务。

不要为了追求“最佳实现”停下整个项目。

任何重构、新依赖、新模型、新数据集，在引入前必须回答：

> 它是否是完成当前 milestone 的必要条件？

如果不是，不做。

---

# 13. 当前不接下游

在第一个 sparse planner 实验成功以前：

不要接：

```text
Kimodo
ARDY
SONIC
```

因为现在要独立验证：

> Sparse Planner 有没有学到环境条件与身体轨迹之间的关系。

第一阶段成功以后，再进入：

```text
Sparse anchors
↓
Kimodo constraint
↓
dense motion
```

这是下一阶段，不是当前阶段。

---

# 14. 当前阶段的最终输出

当前阶段不是一套完整机器人系统。

只需要形成：

```text
Simple obstacle condition
        ↓
Conditional Flow Matching
        ↓
8 × 9 Timed Semantic Anchors
        ↓
3D visualization
```

以及两张最关键的实验结果：

### Figure A

不同 obstacle heights 对应不同 foot trajectories。

### Figure B

相同 obstacle condition 下的多模态 trajectory samples。

只要这两项成立：

> 当前阶段成功。

之后才进入 Kimodo / ARDY motion completion。

