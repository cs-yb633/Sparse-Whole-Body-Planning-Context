# Flow Matching base-model technical evaluation

日期：2026-09-10。结论：**A. Recommend HRI Flow Matching**，具体选择其 `ConditionalUnet1D + global_cond + independent CFM` 核心作为未来 fork 的起点。此处是技术推荐，尚未 fork、接受架构变更或实现 V0。

固定目标仍是 `p(K_future | single box, current G1 root/feet state, goal)`，外部接口为 condition `[B,C]`、target `[B,8,9]`。九个值依次为 root x/y/yaw、left foot xyz、right foot xyz；八个位置对应固定的未来时刻，不是 B-spline 系数。没有加入感知、motion completion、控制或机器人系统。

## 1. Revision、审计范围与证据等级

| Candidate | Repository / actual revision | Local checkout |
| --- | --- | --- |
| HRI | [HRI-EU/flow_matching](https://github.com/HRI-EU/flow_matching/tree/516e8e18875b27741bdbdb8a252e904032c90723), `516e8e18875b27741bdbdb8a252e904032c90723` | [hri_flow_matching](</opt/ext_disk/pub/MYB/Sparse Whole-Body Planning/third_party/hri_flow_matching>) |
| MotionFM | [dongzhuoyao/motionfm](https://github.com/dongzhuoyao/motionfm/tree/08eb88dda8c096396132b9d91deae322f16aa224), `08eb88dda8c096396132b9d91deae322f16aa224` | [motionfm](</opt/ext_disk/pub/MYB/Sparse Whole-Body Planning/third_party/motionfm>) |

两个 checkout 均通过 `git clone --depth 1` 放入 reference 区域；不是我们自己的 fork。开始和结束均检查 `git status --porcelain=v1 --untracked-files=all`。审计未修改 upstream；运行设置 `sys.dont_write_bytecode=True`。

- **verified-static**：实际读过固定 revision 的数据构造、调用点、forward、loss、sampler。下文源码链路和形状推导属于此级别。
- **verified-core-runtime**：实际执行了隔离源码核心，含明示的依赖绕过/测试替身；只有第 9 节列出的命令、数值和 shape 属于此级别。
- **unverified**：两仓库原生完整 training/inference entrypoint、真实数据文件的实际取样、原配置完整网络、checkpoint、生成质量、收敛、物理合理性、跨障碍能力、长期 ODE 稳定性及 V0 实现。没有把这些写成已运行成功。
- **hypothesis / 工程估计**：未来适配量和推荐；由代码证据支持，但不是研究实验结果。

本次没有下载训练集或 checkpoint。训练 sample 的追踪基于**真正由训练入口调用的 Dataset / collate / target construction**，不是论文或 README 中的抽象图；由于未读取原任务数据文件，原任务 sample 的磁盘实测 shape 明确为 **unverified**。随机 `[B,8,9]` 核心测试也不替代该验证。

## 2. Source-code map

以下链接固定到本次 revision；后文 H / M 编号指向这些源码依据。

| ID | Source / class or function | 审计用途 |
| --- | --- | --- |
| H1 | [external/models/pusht.py:613](https://github.com/HRI-EU/flow_matching/blob/516e8e18875b27741bdbdb8a252e904032c90723/external/models/pusht.py#L613), `create_sample_indices`, `sample_sequence`, `PushTImageDataset.__getitem__` | 原始 zarr、归一化、window、padding |
| H2 | [examples/flow_pusht.py:61](https://github.com/HRI-EU/flow_matching/blob/516e8e18875b27741bdbdb8a252e904032c90723/examples/flow_pusht.py#L61), `train`, `test` | horizon、condition、CFM loss、Euler |
| H3 | [external/models/unet.py:31](https://github.com/HRI-EU/flow_matching/blob/516e8e18875b27741bdbdb8a252e904032c90723/external/models/unet.py#L31), `ConditionalUnet1D.forward:219`, `ConditionalResidualBlock1D.forward:110` | 实际 velocity backbone、time embedding、FiLM |
| H4 | [external/models/kitchen_lowdim_dataset.py:15](https://github.com/HRI-EU/flow_matching/blob/516e8e18875b27741bdbdb8a252e904032c90723/external/models/kitchen_lowdim_dataset.py#L15), `KitchenLowdimDataset`；[examples/flow_kitchen.py:74](https://github.com/HRI-EU/flow_matching/blob/516e8e18875b27741bdbdb8a252e904032c90723/examples/flow_kitchen.py#L74) | 最接近 V0 的纯 vector condition 路径 |
| H5 | [external/models/TransformerForDiffusion.py:11](https://github.com/HRI-EU/flow_matching/blob/516e8e18875b27741bdbdb8a252e904032c90723/external/models/TransformerForDiffusion.py#L11)；[examples/flow_pusht_transformer.py:98](https://github.com/HRI-EU/flow_matching/blob/516e8e18875b27741bdbdb8a252e904032c90723/examples/flow_pusht_transformer.py#L98) | 仓库也有 Transformer，避免把候选简化为只有 UNet |
| H6 | [examples/flow_mimic.py:71](https://github.com/HRI-EU/flow_matching/blob/516e8e18875b27741bdbdb8a252e904032c90723/examples/flow_mimic.py#L71)；[RobomimicReplayLowdimDataset._data_to_obs:141](https://github.com/HRI-EU/flow_matching/blob/516e8e18875b27741bdbdb8a252e904032c90723/external/diffusion_policy/dataset/robomimic_replay_lowdim_dataset.py#L141) | 14D→20D rotation conversion、normalizer、提前退出 |
| H7 | [vendored Diffusion Policy ConditionalUnet1D](https://github.com/HRI-EU/flow_matching/blob/516e8e18875b27741bdbdb8a252e904032c90723/external/diffusion_policy/model/diffusion/conditional_unet1d.py)；[resnet.py](https://github.com/HRI-EU/flow_matching/blob/516e8e18875b27741bdbdb8a252e904032c90723/external/models/resnet.py) | 来源、复用边界 |
| M1 | [data_loaders/get_data.py](https://github.com/dongzhuoyao/motionfm/blob/08eb88dda8c096396132b9d91deae322f16aa224/data_loaders/get_data.py)；[Text2MotionDatasetV2:233](https://github.com/dongzhuoyao/motionfm/blob/08eb88dda8c096396132b9d91deae322f16aa224/data_loaders/humanml/data/dataset.py#L233)；[get_opt.py](https://github.com/dongzhuoyao/motionfm/blob/08eb88dda8c096396132b9d91deae322f16aa224/data_loaders/humanml/utils/get_opt.py) | HumanML3D/KIT target、length、normalization |
| M2 | [data_loaders/tensors.py](https://github.com/dongzhuoyao/motionfm/blob/08eb88dda8c096396132b9d91deae322f16aa224/data_loaders/tensors.py)；[a2m/dataset.py:85](https://github.com/dongzhuoyao/motionfm/blob/08eb88dda8c096396132b9d91deae322f16aa224/data_loaders/a2m/dataset.py#L85) | collate / mask / action-conditioned target |
| M3 | [model/mdm_flow.py:10](https://github.com/dongzhuoyao/motionfm/blob/08eb88dda8c096396132b9d91deae322f16aa224/model/mdm_flow.py#L10), `MDM_Flow.forward:278`, `TimestepEmbedder:387`, `InputProcess:407`, `OutputProcess:435` | 实际 model、condition、time、shape |
| M4 | [utils/model_util.py](https://github.com/dongzhuoyao/motionfm/blob/08eb88dda8c096396132b9d91deae322f16aa224/utils/model_util.py)；[model/mdm.py](https://github.com/dongzhuoyao/motionfm/blob/08eb88dda8c096396132b9d91deae322f16aa224/model/mdm.py) | factory 与 MDM 代码关系 |
| M5 | [flow/flow_matching_class.py:30](https://github.com/dongzhuoyao/motionfm/blob/08eb88dda8c096396132b9d91deae322f16aa224/flow/flow_matching_class.py#L30), `masked_l2:61`, `sample_euler_raw:78`, `p_sample_loop:196`, `training_losses:377` | 真实 FM 和 ODE |
| M6 | [training_loop_flow.py:329](https://github.com/dongzhuoyao/motionfm/blob/08eb88dda8c096396132b9d91deae322f16aa224/training_loop_flow.py#L329), `forward_backward`；[train.py:21](https://github.com/dongzhuoyao/motionfm/blob/08eb88dda8c096396132b9d91deae322f16aa224/train.py#L21) | batch→loss→backward、debug overrides |
| M7 | [model/cfg_sampler.py](https://github.com/dongzhuoyao/motionfm/blob/08eb88dda8c096396132b9d91deae322f16aa224/model/cfg_sampler.py)；[config/config_base.yaml:37](https://github.com/dongzhuoyao/motionfm/blob/08eb88dda8c096396132b9d91deae322f16aa224/config/config_base.yaml#L37)；[generate.py:90](https://github.com/dongzhuoyao/motionfm/blob/08eb88dda8c096396132b9d91deae322f16aa224/generate.py#L90) | guidance、solver defaults、generation horizon |
| M8 | [model/rotation2xyz.py:12](https://github.com/dongzhuoyao/motionfm/blob/08eb88dda8c096396132b9d91deae322f16aa224/model/rotation2xyz.py#L12)；[model/smpl.py](https://github.com/dongzhuoyao/motionfm/blob/08eb88dda8c096396132b9d91deae322f16aa224/model/smpl.py) | 强制 SMPL constructor 依赖 |

## 3. Target tensor trace：真正学的是什么

### HRI

**PushT 实际训练路径（H1→H2→H3）：**

```text
zarr data/action [N,2]（二维推杆/agent 位置 action）
  → 按维 min/max normalize 至 [-1,1]
  → 按 episode 取 pred_horizon=16 的 window
  → __getitem__: action [16,2]
  → DataLoader(batch_size=64): x_traj=x1 [B,16,2]
  → CFM: x0/xt/ut [B,16,2]
  → ConditionalUnet1D: [B,16,2] → [B,2,16] → temporal UNet
  → vt [B,16,2] → MSE(vt,ut)
```

轨迹没有 flatten 成 `[B,32]`。`get_data_stats` 中的 `reshape(-1,D)` 只用于统计；训练保留时间轴。`obs_horizon=1`、`action_horizon=8` 不等于生成 horizon：生成 16 步，执行其中 8 步。window 起点含当前 action，不天然等于八个 strictly future anchors；V0 Dataset 必须自己保证语义。

PushT `pad_before=obs_horizon-1=0`、`pad_after=action_horizon-1=7`；越界处复制首/末帧，**没有 loss mask 或 attention mask**，重复尾帧也参与 MSE。图片只截取前一个观察时刻。

**Kitchen（H4）：** `.npy` observations/actions/masks 原文件轴序为 `[T,N,D]`，`transpose_batch_timestep` 转成 `[N,T,D]`，逐 episode 加入 ReplayBuffer，SequenceSampler 取 16 步。训练直接使用 `data['action']`，配置 action_dim=9，所以预期 `x1=[B,16,9]`。条件是 `data['obs'][:,:1]`，预期 `[B,1,60]`。这里 `vision_feature_dim=60` 是变量名，实际没有视觉 encoder。构造出的 `get_normalizer()` **没有被该训练脚本调用**；不能说所有 HRI 路径均归一化。另有 `eps_len=int(masks[i].sum())` 随即被硬编码 `eps_len=409` 覆盖；应绕过此原任务 Dataset，不沿用其 mask 处理。

**Robomimic（H6）：** HDF5 `actions` 的双臂 `[N,14]` 可 reshape 为 `[N,2,7]`，每臂 xyz + axis-angle + gripper 经 rotation-6D 转换成为 10 维，回到 `[N,20]`；配置生成 `[B,16,20]`，obs_dim=50。`normalizers.normalize(data)` 在 loss 前执行。**该 revision 在打印 batch shape 后 `sys.exit(0)`，后续 FM loss 实际不可达**；20D/50D 是配置/转换路径的预期，不是本次数据实测，也不保证任意 HDF5 与默认 obs_keys 相容。

**V0 适配判断：自然。** `input_dim=9`，horizon 取 8，外部张量直接 `[B,8,9]`。三层 UNet 的时间长度为 `8→4→2→4→8`，无需 flatten/reshape 72 维联合向量。八个时刻不作为 Transformer token，但作为 temporal Conv1d 的八个位置显式保留并互相作用。默认三层结构要求选择与降采样/skip 对齐的长度；不能据此宣称任意奇数 horizon 都可无改动工作。固定 8 已通过核心运行验证。

### MotionFM

**HumanML3D 实际训练路径（M1→M2→M6→M5→M3）：**

```text
HumanML3D/new_joint_vecs/<id>.npy [L,263]
  → Text2MotionDatasetV2：caption 关联片段，按 unit_length 裁剪
  → (motion-Mean)/Std；zero-pad 到 [196,263]
  → t2m_collate: b[4].T → [263,196] → unsqueeze → [263,1,196]
  → collate: x_start（数学 x1）[B,263,1,196]
  → FlowMatching.training_losses: xt/velocity target 同 shape
  → InputProcess: [196,B,263] → Linear(263,d) → [196,B,d]
  → 加 condition/time token → Transformer
  → OutputProcess: [196,B,d] → [196,B,263] → [B,263,1,196]
```

263 是 HumanML feature 数，factory 为兼容 MDM 接口把它命名为 `njoints=263,nfeats=1`，**并非 263 个物理关节**。表示含 root yaw velocity、root 平面 velocity、root height、relative joint xyz、6D rotations、local velocities、foot contacts；不是 V0 的 root absolute x/y/yaw + 两脚 xyz。`motion_process.py` 中的 feature concat 和 `get_model_args` 相互印证。

KIT 同路径为 `[B,251,1,196]`。`get_opt` 明确把两者最大长度设为 196，HumanML wrapper 不把训练配置 `num_frames=60` 用于覆盖这个上限；不能给两种数据都套用 60 帧。原长度筛选为 HumanML ≥40 / KIT ≥24 且 <200，随后按 unit_length 裁剪及 padding。

**Action-to-motion 路径（M2/M4）：** HumanAct12 / UESTC pose 转 rot6d，24 个 SMPL pose joints 加一个 translation 槽，translation xyz 后补零到 6 维；默认 shape `[B,25,6,60]`，每时刻实际线性输入宽度 150。frame sampling 在短序列上可重复尾帧；这里也不是 `[B,263,1,196]`。

`collate` 根据 lengths 生成 `mask=[B,1,1,T]`。**FM loss 使用 mask，默认 Transformer forward 没有传 `src_key_padding_mask`**（代码中只有注释），因此 padded tokens 仍会影响有效 token 的 attention。原始 frame count 也不是独立的 horizon condition embedding。

**V0 适配判断：容易，但不是原生 rank-3 接口。** 用 `njoints=9,nfeats=1`，将 `[B,8,9]` 转为 `[B,9,1,8]`；内部只 flatten 每帧 `J×F=9`，得到八个 `[9]` 输入 token，再 Linear 到 d。输出逆变换回 `[B,8,9]`。不需要把 72 个值拆成 72 tokens，也不应把 8 个时刻塞到 joint axis。固定八个未来时刻可使用现有 sequence position encoding；新 Dataset、factory 和 collate 必须解除原有 196/60 及 human feature 假设。

## 4. Backbone、temporal dependency 与来源

| 项目 | HRI 主路径 H3 | MotionFM 常规配置 M3/M4 |
| --- | --- | --- |
| 实际 class | `ConditionalUnet1D(nn.Module)` | `MDM_Flow(nn.Module)` |
| 类型 | temporal 1D UNet；不是逐点 MLP | 默认 model config `trans_enc`，Transformer Encoder；不是 DiT |
| 原配置 | down_dims `[256,512,1024]`，kernel=5，GroupNorm=8 | latent=512，8 layers，4 heads，FF=1024，dropout=.1 |
| 轨迹 embedding | 首个 Conv1d 将 D 映射到 channels，沿 T 卷积 | 每帧 Linear(J×F,d)，每帧一个 token |
| 时间依赖 | 卷积、两次降采样、bottleneck、skip、上采样；不是独立处理各时刻 | 非 causal self-attention 在 frame tokens 和 condition token 之间交互 |
| flow-time t | 连续 sinusoidal(t) 256D → Linear/Mish/Linear | `floor(1000*t)` 查 sinusoidal PE buffer → Linear/SiLU/Linear |
| sequence position | 无显式 absolute timestamp/position embedding；卷积位置、边界和多尺度结构提供时序结构 | 固定 sinusoidal sequence PE，加在 `[T+1,B,d]` 上 |
| output | Conv1d head → `[B,T,D]` velocity | Linear head + reshape → `[B,J,F,T]` velocity |

**MotionFM 与 MDM 的真实关系：** `create_model_and_flow` 实例化 `MDM_Flow`，diffusion factory 另实例化 `MDM`。两类都直接继承 `nn.Module`，不是 `MDM_Flow(MDM)` 的 Python 继承。对同仓库两文件执行 `diff -u model/mdm.py model/mdm_flow.py` 可见复制/修改关系：InputProcess、Transformer encoder/decoder/GRU、OutputProcess、CLIP/action conditioning、Rotation2xyz 及辅助方法骨架沿用；flow 版本支持 scalar ODE time broadcast，将浮点 t 乘 1000 后查表，增加 T5 token conditioning，loss 和 ODE 则转到 `FlowMatching`。这验证的是代码级派生关系；精确对应原 MDM 上游哪个历史 commit **unverified**。

模型也支持 `trans_dec` 和 `gru`，本次推荐对比以通常的 `trans_enc` 为主。**无参数执行 train.py 的实际默认与 model YAML 不完全相同**：`config_base.yaml` 开启 `is_debug=true`，入口强制 `dataset=kit`、`arch=trans_dec`、100 steps 并启用 evaluator。报告中的“默认 encoder”指 model 配置和常规 `is_debug=0` 路径，不是声称裸入口会运行 encoder。

**HRI 与 Diffusion Policy 的真实关系：** 同仓库保留完整 `external/diffusion_policy`，含 temporal UNet / Conv1d / sinusoidal embedding、Transformer、normalizer、ReplayBuffer/SequenceSampler、dataset/env runner。HRI 实际 examples 导入 `external/models/unet.py` 的简化版本：FiLM 固定预测 scale+bias，去掉 vendored 大版本的 local_cond 分支等；没有调用 diffusion scheduler 做训练。ResNet-18 去分类 head 并以 GroupNorm 替换 BatchNorm 的视觉路径也沿用此组件体系。继承的是代码与架构组件，学习目标是 velocity FM，而非 DDPM 噪声目标。vendored Diffusion Policy 的原始独立 git revision 未随本次检查另行核验。

HRI 还有 H5 的 `TransformerForDiffusion`：每个轨迹时刻 Linear→token，learned position embeddings，condition/time memory tokens 经 encoder/MLP 后供 Transformer decoder cross-attention；默认 12 layers / 12 heads / 768D、noncausal。PushT example condition `[B,1,514]`，time token `[B,1,768]`；实际 memory 长度为 2。`n_obs_steps=None` 会把 position buffer 容量按 horizon 分配，但 forward 按实际 token 数切片。其名字含 Diffusion 也不改变外层 CFM velocity 训练。本次没有运行这个分支，推荐也不依赖它。

## 5. Condition trace 与 V0 替换位置

### HRI

```text
PushT raw image [B,1,3,96,96] + agent_pos [B,1,2]
  → flatten B×obs_time: [B,3,96,96]
  → ResNet18(no classifier, GN): [B,512]
  → reshape [B,1,512]，concat normalized agent_pos → [B,1,514]
  → flatten observation-time/features → obs_cond [B,514]
  → concat with flow-time embedding [B,256] → global_feature [B,770]
  → each residual block: Mish + Linear → [B,2*out_channels,1]
  → scale*activation + bias，沿该层全部 temporal positions broadcast
  → velocity [B,16,2]
```

这是 global_cond + FiLM，不是把 observation 直接 append 到每个 action 的输入维，也不是 cross-attention。每个 residual block 有自己的 condition projection；它的 scale/bias 在该层共享于全部时刻。

Kitchen 是 `obs [B,1,60] → flatten [B,60] → concat time [B,256] → [B,316] → FiLM`，**encoder 为 identity/flatten**。Robomimic 预期为 normalizer 后 `[B,1,50]→[B,50]→[B,306]`，但完整训练被上述 `sys.exit` 截断。

审计的 HRI examples 没有 condition dropout、unconditional/conditional 双调用或 CFG。设置 `global_cond=None` 不是已训练的 unconditional 模式：若构造时 `global_cond_dim>0`，线性层仍要求对应宽度；应传相应向量，或单独构造 `global_cond_dim=0` 的无条件模型。不能从 `ConditionalFlowMatcher` 的类名推断存在 classifier-free guidance。

**V0 替换：** 新 Dataset 输出 box geometry + current root/feet + goal 的 `[B,C]`，以 identity 或小 MLP 得到 `[B,C_e]`，构造 `global_cond_dim=C_e` 并在 train / sample 每步传入。删除视觉 encode/flatten 逻辑，UNet 的 forward 和 FiLM 无需改变。C 的字段和数值单位尚未在本次冻结；smoke 的 C=18 是任意测试宽度，不是项目 representation 决策。

论文标题含 affordance，但被追踪的 examples 使用 image/proprioception 或低维 state；没有找到一个必须独立加载的 affordance network 进入这些 velocity calls。不能把标题当成隐藏 encoder 依赖，也不能据此声称已复现论文全部 affordance 功能。

### MotionFM

```text
HumanML/KIT caption：长度 B 的字符串列表
  → CLIP tokenize：截到20词加起止token，再补到 [B,77]
  → frozen CLIP ViT-B/32 encode_text → [B,512]
  → mask_cond (training p=.1; force_mask for CFG)
  → Linear(512,d) → [B,d]
  → 加到 time embedding [1,B,d]
  → prepend joint time+condition token 到 T 个 motion tokens
  → [T+1,B,d] → PE → Transformer Encoder → 去掉首token
  → velocity [B,J,F,T]
```

注意 condition dropout 在 CLIP projection **之前**，因此 zero condition 经有 bias 的 Linear 后不一定为零；CFG 以同一 force_mask 分支定义 unconditional prediction，不能误读为另一个模型。

action 模式：class id `[B,1] → EmbedAction` 表索引得到 `[B,d] → mask_cond → 加到 [1,B,d]`。这是离散类别 embedding，**不能把任意连续向量强塞进 action id**。T5 可选模式：text→30 tokens→frozen T5-large encoder `[B,30,1024]`→Linear `[B,30,d]`→transpose，与一个 time token concat 为 `[31,B,d]`，再与 motion sequence concat；属于额外路径，未实测。

默认 encoder 没有将 condition 在每个时刻重复 concat；所有时刻通过 attention 读取同一个 condition/time token。decoder 变体使用 condition/time 作为 memory；GRU 分支显式 repeat condition 沿时间拼入特征，本次没有运行或推荐该分支。

M7 的 CFG wrapper 对同一 x/t 调用 conditioned 和 `y['uncond']=True`，计算 `v_uncond + scale*(v_cond-v_uncond)`；scale `[B]` reshape 为 `[B,1,1,1]`。CFG 存在，通常每个 ODE evaluation 需要两次网络调用。原 wrapper 断言 cond_mode 只能 text/action，V0 若新增 vector 模式也须更新或先绕过 CFG。

**V0 替换：** `get_model_args` 去掉 dataset→263/251/25×6 硬编码，新增 vector cond 模式与 Linear/MLP(C,d)，从 `y['condition']` 读取 `[B,C]`；修改 `MDM_Flow.__init__/forward` 的条件选择和 `mask_cond` 调用，保留 `emb += condition_embedding` 与 encoder token 路径。移除 CLIP/T5 imports/loaders，解除 `Rotation2xyz` constructor 及 `_apply/train` 的强引用。用新 collate 提供 `[B,9,1,8]` 和全真 mask。不是只改 config 就能原生支持 continuous condition。

## 6. Flow Matching 数学与真实 loss path

### HRI：independent conditional flow matching，sigma=0

H2 `train:125–141` / H4 `129–140` 明确使用 TorchCFM 的 `ConditionalFlowMatcher(sigma=0.0)`；不是 `ExactOptimalTransportConditionalFlowMatcher`，没有 minibatch OT coupling。

依赖源码本次固定为下载的 **torchcfm 1.0.7**，查看并执行 [conditional_flow_matching.py](</opt/ext_disk/pub/MYB/Sparse Whole-Body Planning/artifacts/flow_base_audit/deps/extracted/torchcfm/conditional_flow_matching.py:40>) 中 base class 和 `pad_t_like_x`。HRI requirements 未固定其版本，所以这不是声称 upstream 历史训练一定使用 1.0.7。

| 变量/步骤 | 真实函数/语句 | V0 shape |
| --- | --- | --- |
| x1 | Dataset action → `x_traj.float()` | `[B,8,9]` |
| x0 | `torch.randn(x_traj.shape,device=...)` | `[B,8,9]`, N(0,I) |
| t | `sample_location_and_conditional_flow`: `torch.rand(B).type_as(x0)` | `[B]`, U[0,1) |
| broadcast t | `pad_t_like_x` | `[B,1,1]` |
| path | `compute_mu_t`: `(1-t)*x0+t*x1`; `sample_xt` 加 sigma*eps | `[B,8,9]`；sigma=0 时精确线性 |
| target velocity ut | `compute_conditional_flow`: `x1-x0` | `[B,8,9]` |
| predicted vt | `noise_pred_net(xt,t,global_cond=obs_cond)` | `[B,8,9]` |
| loss | `torch.mean((vt-ut)**2)` | scalar，全部时刻/特征参与 |

外部 semantic condition 不参与 `xt/ut` 的随机配对构造，进入 velocity network；每个训练 x1 的对应 condition 保持一致。sigma=0、independent noise-data pairing 的直线路径是真正的 FM，可作 rectified-flow 风格最小基线；无 reflow / OT 实验声称。normalization 是 Dataset/调用者责任，见第 3 节分支差异。

### MotionFM：Gaussian conditional path，sigma_min=1e-4

M6 `forward_backward` 调用 M5 `training_losses(model,micro,t=None,model_kwargs=micro_cond,...)`，再 `loss.mean()` 和 backward。`x_start` 在此表示 data endpoint，即本报告数学 x1，不能因 diffusion 命名误当成噪声。

| 变量/步骤 | M5 training_losses 的真实语句 | V0 internal shape |
| --- | --- | --- |
| x1 | `x_start` | `[B,9,1,8]` |
| x0 | `noise=torch.randn_like(x_start)`，或调用者传 noise | `[B,9,1,8]`, N(0,I) |
| t | `assert t is None; t=torch.rand(len(x_start),...)` | `[B]` |
| broadcast t | `t=t[:,None,None,None]` | `[B,1,1,1]`，源码固定 4D |
| xt | `t*x_start+(1-(1-sigma_min)*t)*noise` | `[B,9,1,8]` |
| target velocity | `x_start-(1-sigma_min)*noise` | `[B,9,1,8]` |
| predicted velocity | `model(x_t,t_1d,**model_kwargs)` | `[B,9,1,8]` |
| loss | `masked_l2(target,model_output,mask)` → `terms['rot_mse']` | `[B]`；trainer mean 得 scalar |

这里 sigma(t)=1−(1−epsilon)t，t=1 终点为 `x1+epsilon*x0`，不是 HRI sigma=0 的精确 endpoint。可以描述为 conditional Gaussian / OT-style straight conditional path；**没有数据 minibatch OT 配对求解**。不依赖 TorchCFM，公式直接写在本仓库。名称 `rot_mse` 和残留 diffusion docstring 不改变它实际预测 velocity 的事实。

`masked_l2=sum((target-v)^2*mask)/(J*F*sum(mask))`，每样本归一化；全零 mask 会除零，固定 8 步用全真 mask。原公式 t broadcast 写死 4D，若坚持直接传 `[B,8,9]` 而不做 adapter，会出现不正确广播/shape；保留 `[B,9,1,8]` 最省修改。

默认 `lambda_rcxyz=lambda_vel=lambda_fc=0`，loss 只有上述 velocity MSE。额外 xyz/foot-contact/skeleton loss 可完全绕过，不该带入九维 anchors。`model_kwargs['y']['mask']` 在 None guard 前访问，因此即便 `no_cond` 仍需至少提供 y/mask 字典。HumanML/KIT 标准化在 Dataset 完成，FM 类不自行 normalize；action path 转 rot6d / root-relative translation，也没有同样的 HumanML Mean/Std 标准化。

## 7. Sampling / ODE trace

### HRI

H2 `test:211–223`、H4 `202–214` 是手写 explicit Euler：

```text
noise → z0 [B,T,D]
for i=0..N-1:
    t = tensor([i/N])
    v = ConditionalUnet1D(z,t,global_cond=c)
    z = z + v/N
unnormalize（仅实际使用 normalizer 的分支）→ trajectory → 原任务 action chunk
```

**实际默认步数按入口区分**：PushT UNet=1，Kitchen=16，Robomimic=1，PushT Transformer=16。不是 RK4，也不是调用 torchdiffeq；虽有这些 imports，不能据此认定被使用。

**必须修正的源码问题：** 所有这些示例 sampling 使用 `torch.rand`（U[0,1)），training 是 `torch.randn`（N(0,I)）。这会改变起始分布，不能把示例视为概率路径一致的现成 sampler。应在未来 fork 使用一次 `randn(B,8,9)` 初始化；不在循环里反复创建无用 noise。当前 smoke 已**显式修正此点**，因此是适配核心 Euler 检查，不是上游 test() 原样运行成功。

condition 在 ODE 外编码一次，每步显式传同一个 `global_cond`。rank/shape 始终 `[B,T,D]`。原测试硬编码单环境、`noise` 的 leading dim=1，再 `expand(x_img.shape[0],...)`：这里 observation time 不是候选数量，expand 也不是独立多候选噪声。velocity 核心支持 batch；V0 只需重复每个 condition 并独立采样 batch noise，去掉 environment 和 action chunk wrapper。生成 `[8,9]` 无需改 Euler 更新公式，只改 shape/config/wrapper。

建议 V0 若之后获准实现，从固定小步数 Euler 开始，并用 float Tensor 传 t。H3 中 Python 非 Tensor 的 float 会被构造成 long tensor；直接传 `0.25` 会丢掉小数，应避免或修正此转换。当前 smoke 传浮点 Tensor。

### MotionFM

M5 `p_sample_loop`：

```text
torch.randn(*shape) → z0 [B,J,F,T]
func = lambda t,x: model(x,t,**model_kwargs)
odeint_adjoint(func,z0,tensor([0.,1.]),adjoint_params=(),...)
  → solver-state stack [2,B,J,F,T]
  → [-1] 得 [B,J,F,T]
  → V0 adapter 回 [B,8,9]
```

M7 base config 使用 `method=euler,step_size=.01`，对应区间 [0,1] 上 **100 Euler steps**。`dopri5` 是显式支持的 adaptive 选项，rtol=atol=1e-5，没有固定 100 次 evaluation 的承诺；debug/generation modes 会覆盖 solver。`sample_euler_raw` 也有手写 Euler，N=`int(1/step_size)`；不是 RK4。editing/inpainting 分支不是 V0 必需，未运行。

`model_kwargs` 在 closure 内每次传入。文本 encoder 在 `MDM_Flow.forward` 中调用，因此原路径会在每个 evaluation 重算 text encoding；CFG 时调用两次模型。未来 vector embedding 可缓存或每次算小 MLP，condition 内容必须一致。M3 支持 scalar ODE t broadcast，但 embedding 将 t 量化成 `floor(1000*t)`，不是精确连续 time encoding；adaptive solver 的精度/稳定性不能仅凭 tolerances 保证。

batch dimension 支持多候选，`generate.py` 还支持 num_repetitions。原生成入口会按 human dataset hardcode 196/60 上限、fps、motion_length，并做人形反归一化/xyz/rendering，不能原封不动用于 V0。保留 4D adapter 时，loss 和 solver 核心均无需修改 horizon-specific 算法；修改 shape construction 和移除人形后处理即可。若改成 rank-3，还须修 loss time broadcast、CFG scale 和部分 editing reshape。

## 8. Dependency / engineering burden

统计来自两个 pinned checkout 的 `git ls-files`，包括 vendored 源码，不含生成文件：

| Inventory | HRI | MotionFM |
| --- | --- | --- |
| tracked files | 543 | 129 |
| Python files | 247 | 85 |
| Python physical lines（含注释/空行） | 60,206 | 18,481 |
| 最小拟保留计算核心 | 一个自包含 temporal UNet 的 classes + CFM base + Euler | MDM_Flow 的 embedding/Transformer/head + FM loss + solver |

这不是“代码少者必胜”的度量。HRI 总仓库更大，但大部分 vendored code 可以整体绕过；MotionFM 文件更少，模型构造与人形资产的耦合更直接。实际原生 import probe 分别在 HRI `unet.py:9 import zarr` 和 MotionFM `mdm_flow.py:6 import clip` 失败，详见 log。

### HRI

| 依赖 | 实际引用位置/角色 | V0 必要性与剥离方法 |
| --- | --- | --- |
| torch、Python math/typing | H3 所有实际 UNet classes | 核心必需 |
| TorchCFM | examples 的 `ConditionalFlowMatcher` | 若保留原训练调用则需；base sigma=0 只需 torch。OTPlanSampler/POT 并未被该 matcher 使用，可通过聚焦导入/提取 base 解除模块级 import 负担 |
| numpy | Dataset / stats | 数据与normalization用；H3 class 数学不依赖其计算 |
| Diffusion Policy external | Dataset、sampler、normalizer、模型祖先、env | **完整包非必需**；保留模型来源通知，提取自包含 H3 或少量所需组件即可 |
| diffusers | EMA、LR scheduler；H3 顶层也导入 DDPM scheduler | EMA/scheduler 可选；DDPM 不参与 FM。清理 H3 未用 imports 后可不装 |
| ResNet/torchvision/image encoder | PushT image condition | 当前纯向量输入不需要 |
| zarr、gym、pygame、pymunk、shapely、OpenCV、skimage、skvideo、IPython、gdown | H3 顶层混入大量未被 class 使用的 imports；PushT data/env/render | 原任务专属，需清理 imports，不能只是不调用 env |
| Robomimic、h5py、rotation conversion、Franka/MuJoCo | manipulation dataset/env、abs action | V0 整体绕过；HRI vendored Robomimic 不必安装 |
| affordance model | 本次追踪路径中没有独立 loader/encoder | 无已建立的必要依赖 |
| torchdiffeq、torchsde、torchdyn、Lightning、quality-check packages | requirements/若干 examples imports | 当前手写 Euler + base CFM 不需要这些 solver/framework |

HRI `requirements.txt` 未 pin 多个关键包；安装全表不是本次最小测试的前提。`unittest.sh` 只是设置路径后调用 `flow_pusht.py unittest`，脚本的顶层 Dataset/model 初始化仍在发生，打印成功不构成独立 forward/backward 测试。本次未用该输出充当证据。

### MotionFM

| 依赖 | 实际引用位置/角色 | V0 必要性与剥离方法 |
| --- | --- | --- |
| torch、numpy、einops | M3 网络和 shape/PE；M5 数学 | 保留核心源码时需要；numpy 主要为常量，einops 处理维序 |
| torchdiffeq | M5 顶层 `odeint_adjoint` 与 p_sample_loop | 保留此默认 sampler 需该小依赖及其 numpy/scipy 安装依赖；若仅保留手写 Euler 可去掉它 |
| HumanML3D、KIT-ML、GloVe/WordVectorizer、spacy | M1 数据格式、文本/evaluator | 不是 FM 数学必需；替换整个 data-loader factory/collate 后可删除；不能继续使用其 263/251D representation |
| CLIP | M3 顶层 import，text constructor/load 和每次 forward | 连续向量条件不需要；须改 import、condition 构造和 forward，单设 no_cond 仍不能消除顶层 import |
| T5 / transformers / sentencepiece | 可选 text encoder | 删除 text path 后不需要，也不下载权重 |
| SMPL / smplx | M3 无条件 `Rotation2xyz(...)`，M8 再无条件 `SMPL()`；`_apply/train` 强引用 | 语义上非必需，但目前是构造期硬耦合；即使 xyz 或 no_cond 也会实例化，必须移除/延迟 constructor 及辅助引用 |
| PyTorch3D | `utils/rotation_conversions.py` 带 PyTorch3D 来源头与独立 BSD license | 该主路径使用拷入的 tensor conversion functions，不是必须安装整套 compiled PyTorch3D；V0 不做 rotation/SMPL 可连此模块也绕过 |
| human-motion evaluator、SMPL rendering / joints2smpl、chumpy | TrainLoop 顶层 import evaluator、生成时 xyz/render | V0 全部不需要；仅关 eval flag 不能消除顶层 import，需拆 training/generate shell |
| wandb、ClearML、blobfile、Hydra、distributed helpers | train.py / TrainLoop / M5 顶层 wandb 和 dist_util，后者 import blobfile | logging/config/训练基础设施，不是 FM 必需；删 telemetry 与对应顶层 imports，简化入口即可 |
| diffusion package | M5 只需 `diffusion.nn.sum_flat`；factory 同时导入 diffusion model | 该 helper 可保留，整套 GaussianDiffusion/SpacedDiffusion 不需要；分离 factory imports |

SMPL **代码依赖**、SMPL **模型资产**与主仓库 license 是不同事项；此次均未下载资产。已检查 license 文件作许可清单，不作额外资产使用权推断。MotionFM MIT 不意味着外部人体数据/SMPL资产自动具有相同许可。

## 9. 最小 smoke-test evidence

脚本：[scripts/audit_flow_bases_smoke.py](</opt/ext_disk/pub/MYB/Sparse Whole-Body Planning/scripts/audit_flow_bases_smoke.py>)。
完整输出：[smoke.log](</opt/ext_disk/pub/MYB/Sparse Whole-Body Planning/artifacts/flow_base_audit/smoke.log>)；机器可读结果和 source SHA256：[smoke_summary.json](</opt/ext_disk/pub/MYB/Sparse Whole-Body Planning/artifacts/flow_base_audit/smoke_summary.json>)。

实际命令（从主 workspace 运行；Python path 是现有环境，不启动 Isaac）：

```bash
envs/eirl_main/bin/python -m pip download --no-deps torchcfm==1.0.7 torchdiffeq==0.2.5 -d artifacts/flow_base_audit/deps
envs/isaaclab/bin/python -B scripts/audit_flow_bases_smoke.py > artifacts/flow_base_audit/smoke.log 2>&1
```

两个 wheel 共 **62,782 bytes**；用 Python zipfile 解压到 artifacts 中，由脚本加入 sys.path，未安装到现有环境。首次尝试用 isaaclab Python 调 pip 发现该环境没有 pip，改用已有 eirl_main 的 pip 仅下载。torchcfm=1.0.7，torchdiffeq=0.2.5，PyTorch=2.7.0+cu128；实际只用 **CPU、2 threads、seed=20260910、B=2、C=18**。中途运行权限环境切换后 bwrap 初始化失败，后续本地命令经自动审批在 sandbox 外执行；未扩大测试内容。

### 同一验证标准与执行范围

两边都执行：random condition `[2,18]`、target `[2,8,9]` → forward → 真实 FM loss → backward → AdamW 一步 → 相同四步 Euler → finite/shape assertions；还验证同噪声改 condition 会改变输出、某时刻输出对别的时刻输入梯度非零。后两个检查只证明代码连接存在，不证明学到了障碍规律或多模态。

**依赖处理严格明示：**

1. HRI：AST 读取 `unet.py` 中 class definitions，body 原样执行，只不执行混入的原任务 imports；TorchCFM 同样只执行原样 base class 和 `pad_t_like_x`，绕过未使用的 OT/POT import。down_dims 减为 `[32,64,128]`、time embedding=32，保留三层结构。sample 使用高斯初始化的四步手写 Euler，这是已指出的必要修正，未调用原环境 test()。
2. MotionFM：AST 原样执行 MDM_Flow/embedding/head classes，以及完整 FlowMatching class 和原 `sum_flat`；跳过未使用 import。用明确的测试子类 override CLIP loader/encode_text，把 `[B,C]` 当作 text-feature boundary 输入，`clip_dim=C`，实际保留并运行 Linear projection + time-token addition。Rotation2xyz 使用测试替身，只提供生命周期需要的空 nn.Identity，若调用几何转换会立刻报错。测试设 latent=32、2 layers、FF=64、4 heads、dropout=0、cond_mask_prob=0、全真 mask、辅助人体 loss=0；没有声称真实文本/人体链路被执行。
3. MotionFM 实际使用下载的 torchdiffeq Euler，通过 upstream `p_sample_loop`；额外验证其结果与 upstream `sample_euler_raw(N=4)` 在 atol=1e-6 内一致。HRI/ MotionFM 使用各自原始 FM 公式，没有统一改写成自建 FM。
4. 以上是**隔离核心适配 smoke test**。原生 import 失败以及完整 entrypoint 的 **unverified** 状态保留；不能把测试替身当作移除 SMPL/CLIP 后的正式实现。

| 验证项 | HRI | MotionFM |
| --- | --- | --- |
| condition / external target | `[2,18]` / `[2,8,9]` | 相同 |
| velocity input / output | `[2,8,9]` / `[2,8,9]` | `[2,9,1,8]` / `[2,9,1,8]` |
| internal observed | `[2,9,8]→[2,32,8]→[2,64,4]→[2,128,2]` | `InputProcess=[8,2,32]`；加 token 后 `[9,2,32]` |
| loss | 1.6853302717 | 1.9572790861 |
| loss / gradients finite | true / true，148 个 gradient tensors | true / true，34 个 gradient tensors |
| maximum checked parameter update | 0.0001000389 | 0.0001003146 |
| Euler steps | 4 | 4 |
| sampled external shape / finite | `[2,8,9]` / true | `[2,8,9]` / true |
| same-noise changed-condition max difference | 0.6093825102 | 0.0939853489 |
| cross-time input gradient max | 0.0391765162 | 0.0177229065 |
| native model import | failed: missing zarr | failed: missing clip |
| native full execution | **unverified** | **unverified** |

两测试模型参数量不同（1,060,073 vs 20,425），初始化和 dropout 设置也不同；**loss、condition difference、参数量不用于性能排名**，不是训练公平性 benchmark。目的是相同接口/计算检查，不是比较精度、速度、收敛或样本质量。没有 GPU timing 或大模型原配置测试。

## 10. V0 adaptation map（仅方案，未实现）

### 如果 fork HRI，最少怎么改

**保留：** H3 的 `SinusoidalPosEmb`、Downsample1d/Upsample1d、Conv1dBlock、ConditionalResidualBlock1D、ConditionalUnet1D **class body 可原封不动保留**；删无用 import 后独立使用。保留 TorchCFM base API/公式、velocity MSE、Euler update equation。主模型配置 `input_dim=9,global_cond_dim=C_e`，使用八步 horizon。time tensor 调用规则必须明确，若支持 Python float 则需修正该转换。

**修改：** 原 example 的数据/condition 读取与 train shell；去掉视觉 encoder 和原任务 environment 初始化。将采样噪声改为 randn、shape 改为 `[B,8,9]`、每个候选独立噪声并对齐 condition batch；统一 train/sample normalizer，返回整个八步 sparse trajectory，不执行 action_horizon slicing。固定未来时刻由 Dataset/representation 保证，不复用当前 action window 语义。

**删除/绕过：** PushT/Franka/Robomimic Dataset 和 env、图像输入、完整 vendored training workspaces、DDPM scheduler、unnecessary solver imports、affordance-paper 原任务脚本；不保留 Kitchen 硬编码 episode length 或 Robomimic debug exit。

**新增（之后才写）：** `synthetic_box_dataset.py`、`anchor_representation.py`、`simple_condition_encoder.py`（identity 或小 MLP）、轻量 normalization、minimal train/sample entrypoints。可选的可视化用于原 V0 milestone；本次没有创建这些文件。

### 如果 fork MotionFM，最少怎么改

**保留：** M3 的 PositionalEncoding、TimestepEmbedder、InputProcess/OutputProcess 的现有线性路径、TransformerEncoder 构造和主要 token forward；M5 默认辅助权重为零的 training_losses/masked_l2，以及 p_sample_loop 的 Euler 分支或 sample_euler_raw；M2 通用 collate/lengths_to_mask 若需要变长。采用 `[B,9,1,8]` 内部接口以保留现有 4D FM loss。

**修改：** `MDM_Flow.__init__/forward` 增加 vector condition；去掉 CLIP/SMPL import、constructor 和 `_apply/train` 强引用（顺便恢复标准 nn.Module 返回行为）。`utils/model_util.get_model_args/get_cond_mode` 支持 semantic feature dims 和 C。替换 HumanML collate/mean-std path；拆开 TrainLoop 的 evaluator/logger 以及 generate.py 的 skeleton/frames/fps/rendering。若使用 CFG，更新其 mode assertion 和属性代理，否则直接绕过。

**删除/绕过：** HumanML/KIT/action Dataset、CLIP/T5、SMPL/SMPL-X资产、rotation/xyz recovery、人体 evaluator/contact losses、editing/inpainting、DDPM factory、WandB/ClearML/DDP full training infrastructure。移除的是与当前 planner 无关的内容，不代表这些依赖在原论文任务中无用。

**新增（之后才写）：** 与 HRI 相同的 synthetic Dataset / semantic representation / vector encoder / normalization / minimal train/sample；另需 `[B,8,9]↔[B,9,1,8]` layout adapter，更新模型 condition API。

**工程量估计（hypothesis）：** 两者共同必须新增项目 Dataset、语义表示和训练入口；这部分不能计为某候选的独有缺点。HRI 的候选特有修改集中于 import 清理、sample 修正和 wrapper，核心 UNet forward 不变，难度低到中。MotionFM 多出 constructor 生命周期、condition API、factory 和 4D layout 耦合清理，难度中等。没有给未经实际实现验证的精确行数/工时，也没有声称任何上游预训练 checkpoint 可直接迁移到 G1 anchors。

## 11. 公平对比表

| 指标 | HRI Flow Matching | MotionFM |
| --- | --- | --- |
| 原任务 | PushT、Franka Kitchen、Robomimic robot action sequence generation | text/action-conditioned human motion synthesis/editing |
| Target tensor | PushT `[B,16,2]`；Kitchen 配置 `[B,16,9]`；Mimic 预期 `[B,16,20]` 但训练提前退出 | HumanML `[B,263,1,196]`；KIT `[B,251,1,196]`；action 默认 `[B,25,6,60]` |
| Backbone | 主路径 ConditionalUnet1D；另有 TransformerForDiffusion 分支 | MDM-derived Transformer Encoder，另支持 Decoder/GRU；裸 debug 入口覆盖成 Decoder |
| Temporal modeling | temporal conv + down/up sampling + skips | frame-token self-attention + sinusoidal position |
| Conditioning mechanism | `[B,C]` global_cond，与 t concat 后各层 FiLM broadcast；Transformer 分支 cross-attention | text/action embedding 与 t 相加作一个 token；T5 多 tokens；decoder memory |
| FM implementation | TorchCFM independent CFM，sigma=0，ut=x1−x0 | 自写 Gaussian conditional path，sigma_min=1e-4，ut=x1−(1−eps)x0 |
| Sampling | 手写 Euler，1/16 步视入口；**需修 rand→randn** | torchdiffeq Euler 默认100步；Dopri5 / raw Euler 可选；无训练/采样噪声分布不一致 |
| `[B,8,9]` compatibility | 原生 rank-3，8 与 temporal pyramid 对齐；已核心实测 | 用 `[B,9,1,8]` adapter，自然保留8个tokens；已核心实测 |
| 简单 vector condition compatibility | 原生 global_cond；Kitchen 已直接用状态向量 | 需替换 text/action condition API；测试通过文本 feature 边界替身验证 |
| Human-specific baggage | 主核心无 | CLIP/T5、SMPL constructor、HumanML/KIT representations、evaluator；可删除但有耦合 |
| Robot-specific baggage | vendored DP、PushT/Franka/Robomimic、图像/env imports；大部分易整体绕过 | 主核心无机器人依赖 |
| Dependency complexity | 完整 repo 高；提取 H3+CFM 后低 | 完整入口高；提取核心后低到中，constructor清理多一些 |
| Code modularity | 独立 UNet 好；example train/env 和 unused imports 混杂，sampler有缺陷 | model/dynamic 分文件；model 与 SMPL、train 与 evaluator 强耦合 |
| Smoke-test status | isolated core pass；原生完整执行 **unverified** | isolated core pass，明示文本/SMPL替身；原生完整执行 **unverified** |
| Required changes | import清理、condition/data shell、normalization、Gaussian/batched sampler | vector API、SMPL lifecycle removal、factory/collate/layout、training/render shell |
| License | 主仓库 BSD-3-Clause；Diffusion Policy / TorchCFM MIT，保留来源通知 | 主仓库 MIT（Guy Tevet）；copied PyTorch3D utilities BSD；不自动涵盖外部数据/SMPL资产 |
| Estimated fork difficulty | 低到中（工程估计） | 中（工程估计） |

许可依据：[HRI LICENSE.md](https://github.com/HRI-EU/flow_matching/blob/516e8e18875b27741bdbdb8a252e904032c90723/LICENSE.md)、[Diffusion Policy LICENSE](https://github.com/HRI-EU/flow_matching/blob/516e8e18875b27741bdbdb8a252e904032c90723/external/diffusion_policy/LICENSE)、[MotionFM LICENSE](https://github.com/dongzhuoyao/motionfm/blob/08eb88dda8c096396132b9d91deae322f16aa224/LICENSE)、[copied PyTorch3D LICENSE](https://github.com/dongzhuoyao/motionfm/blob/08eb88dda8c096396132b9d91deae322f16aa224/utils/PYTORCH3D_LICENSE)。TorchCFM wheel 内保留 MIT 许可。未以论文名称、作者领域、star 或论文影响力打分。

## 12. Base Model Recommendation

**A. Recommend HRI Flow Matching.** 推荐的是已审计的 temporal UNet + vector FiLM + CFM 核心，以及后续的小范围 fork 适配；不是认可原 examples 可以原样复制运行。

按用户指定优先级逐项给出理由：

1. **生成 `[B,8,9]`：** HRI 原生接受该 shape，固定8时刻已实测；MotionFM 同样可用，但须4D adapter，原 dataset/factory 更偏人形 representation。HRI 小幅占优。
2. **时间依赖：** 两者都合格，有源码与 cross-time gradient 证据。Transformer 的显式 absolute PE 有价值，但本任务没有“必须 Transformer”约束；不能因 human motion 名称就优先 MotionFM。
3. **simple vector condition：** HRI 原生 global_cond 接口加 Kitchen 已有 identity 状态向量路径，直接对应 V0；MotionFM 需改 forward/constructor 的 condition API。这是主要区分点。
4. **真正的 FM：** 两者均是，具体 path/velocity target 已追踪并运行验证；TorchCFM 包与自写实现不构成优劣的充分理由。
5. **sampling：** MotionFM 的 Gaussian→Euler 默认路径更一致，HRI 原例噪声分布错误是明确缺点。不过修正初始化和 batch wrapper 后，Euler 公式简单，四步 shape/finite test 通过。长期稳定性两者均 **unverified**，不宣称 HRI 更稳定。
6. **修改量：** 共同的新数据/语义模块之外，HRI 无需改变 backbone 的 vector condition API；MotionFM 要解除多处人形/文本强引用。HRI 较少。
7. **剥离依赖：** HRI 全 repo 虽更大，但 H3 class body 自包含，imports 清理即可；MotionFM 的依赖进入 constructor/lifecycle，需更深入改动。HRI 较直接。
8. **license：** 两个主仓库均提供宽松许可，保留各来源通知；没有 license 阻止选择其中一个的源码证据。不是本次决胜因素。
9. **论文影响力：** 未量化或用于排名。

因此没有证据支持 C（两个都不适合）。选择 B 也可实现 V0，且它在 frame-token position encoding 和完整 ODE API 上有优势；当前固定8步、低维向量条件的优先级下，收益不足以抵消额外适配工作。

**停止点：** 本任务到 technical evaluation、文档与 Context Repo 更新为止。未创建 planner、未开始 fork、未训练研究模型、未写 EXP result。`CURRENT_MAINLINE.md` 与 `CURRENT_PHASE.md` 保持原样；SanD Phase C 等此前未完成工作仍保留为未完成，不在本任务中自动执行。
