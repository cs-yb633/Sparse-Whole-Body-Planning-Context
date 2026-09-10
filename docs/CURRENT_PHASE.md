# 当前阶段任务目标：实现并验证 HRI-based Sparse Planner V0

更新时间：2026-09-10

> 本文件是当前执行约束，优先级高于 `ROADMAP.md`。长期方向以 `CURRENT_MAINLINE.md` 为准。

## 0. 当前阶段一句话目标

当前只回答：

> **基于 HRI Flow Matching 的 ConditionalUnet1D + vector global_cond/FiLM + independent CFM，能否根据简单 box 条件生成合理的 G1 `8×9` Timed Semantic Anchors？**

当前不是为了完成完整 G1 系统。

---

## 1. 当前固定任务

学习：

$$
p(K_{future}\mid E,S_{G1},G)
$$

其中：

```text
E = single primitive box geometry
S_G1 = current root/feet state
G = goal
K_future = 8 × 9 Timed Semantic Anchors
```

输出固定为：

```text
root_x, root_y, root_yaw
left_foot_x, left_foot_y, left_foot_z
right_foot_x, right_foot_y, right_foot_z
```

8 个位置对应固定未来时刻，不是 B-spline coefficients。

---

## 2. 当前实现底座

当前实现起点固定为：

```text
HRI-EU/flow_matching
→ ConditionalUnet1D
→ vector global_cond / FiLM
→ independent Conditional Flow Matching
→ Euler sampler
```

HRI 只是 implementation base，不改变研究主线。

正式实现必须修复 HRI example 的 sampling 初始分布问题：训练与采样统一使用 Gaussian `randn`。

MotionFM 仅保留为 Transformer architecture reference，不作为当前首选 base。

---

## 3. 历史 reference repo 不再构成当前 phase

以下工作已不再作为当前执行顺序：

- FlowMP：历史 FM 学习参考，已完成，不再继续；
- ABPolicy：历史 B-spline/refitting 参考，已完成，不再继续；
- SanD-Planner：未来 safety / scene / replanning 可选参考，不阻塞当前 V0。

不得因为这些旧 reference 文档重新回到 Phase A/B/C 顺序。

---

## 4. 当前 V0 输入

只使用简单连续向量：

```text
box center xyz
box size xyz
current root x/y/yaw
current left foot xyz
current right foot xyz
goal x/y/yaw
```

允许使用 identity 或小 MLP condition encoder。

当前禁止加入：

```text
point cloud
depth
voxel
ESDF encoder
PointNet
Point Transformer
VLM/VLA
```

---

## 5. 当前 V0 数据

使用 synthetic expert data。

第一版只需要单一 traversal strategy，例如 left-foot-first。

程序化 expert 必须满足：

$$
z_{foot,max} > h_{box} + margin
$$

当前目标不是数据真实性，而是验证 obstacle condition 是否真的影响 sparse trajectory。

---

## 6. 当前必须实现的最小代码

主 workspace 中实现自己的模块，例如：

```text
src/sparse_planner/
├── anchor_representation.py
├── synthetic_box_dataset.py
├── condition_encoder.py
├── conditional_unet1d.py
├── flow_matching.py
├── train.py
├── sample.py
├── evaluate.py
└── visualize.py
```

不要在 `third_party/hri_flow_matching` 中直接开发；upstream reference 保持 clean。

只提取 HRI 必需的 temporal UNet 核心，并保留来源/revision/license attribution。

---

## 7. Flow Matching 固定为最小版本

采用：

```text
x0 ~ N(0,I)
t ~ U(0,1)
xt = (1-t)x0 + t x1
ut = x1 - x0
vt = v_theta(xt,t,c)
loss = MSE(vt,ut)
```

当前不加入：

- OT-CFM；
- second-order flow；
- reflow；
- RK4；
- 复杂 guidance；
- solver 大规模比较。

采样先用 Euler。

---

## 8. 当前第一个正式实验：EXP-001

固定：

```text
same current G1 state
same goal
same box position/width/depth
```

改变：

```text
box height
```

至少测试：

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

必须输出：

- predicted max left/right foot height；
- required clearance；
- anchor error；
- `box height vs predicted max foot height` 图；
- 不同障碍高度的 anchor trajectory 可视化。

---

## 9. 必须做 condition sanity check

使用相同 initial noise、current state、goal，只改变 box height，确认输出发生有意义变化。

至少增加一个简单 condition ablation，如：

```text
shuffle box condition
```

或固定/置零 box height。

该实验用于确认网络没有忽略 obstacle condition。

---

## 10. 多模态不是当前第一步

EXP-001 成功后，再构造：

```text
left-foot-first
right-foot-first
```

并通过不同 noise sample 验证多模态。

在 EXP-001 之前，不把多模态和基础 obstacle response 混在一起。

---

## 11. 当前明确不接下游

在 EXP-001 成功以前，不接：

```text
ARDY
Kimodo
SONIC
GEAR-SONIC
Isaac full simulation
real robot
```

因为当前必须独立验证：

> `scene geometry → sparse semantic body trajectory` 是否成立。

---

## 12. 当前阶段完成标准

以下全部满足才算完成：

- 自己的 `[8,9]` Timed Semantic Anchor representation 已实现；
- synthetic box dataset 可复现；
- HRI-derived temporal UNet core 已干净抽取；
- vector condition 能通过 FiLM 进入 velocity network；
- CFM forward/loss/backward/sampling shape 均为 `[B,8,9]`；
- 训练可收敛到可用结果；
- EXP-001 显示 obstacle height 增大时 foot anchor height 明确增大；
- condition ablation 证明模型实际利用 obstacle information；
- 有 Figure A / Figure B；
- 真正执行 EXP-001 后更新 `EXPERIMENT_LOG.md`。

---

## 13. 当前阶段禁止钻牛角尖

如果不阻塞 EXP-001，不要停下来研究：

- UNet vs Transformer 谁最终最好；
- anchor 数量最优值；
- Euler vs RK4；
- PointNet / Point Transformer；
- ARDY vs Kimodo；
- G1 全碰撞几何；
- 真机部署；
- 大规模数据；
- 新 foundation model。

当前原则：

> **先证明简单 obstacle condition 能控制 sparse semantic anchor generation。**

---

## 14. 当前阶段最终输出

```text
Single box + current G1 root/feet + goal
                  ↓
HRI-based Conditional Flow Matching Planner
                  ↓
        8 × 9 Timed Semantic Anchors
                  ↓
              Visualization
```

当前最关键结果只有一个：

> **模型是否真正学会根据障碍几何调整未来 semantic body-part anchors。**
