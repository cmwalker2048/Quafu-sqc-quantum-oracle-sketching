# $P\times M$ Parallel QOS ensemble 真机运行指南

对应 Notebook：

`notebooks/14_parallel_qos_ensemble.ipynb`

## 实验矩阵

Notebook 同时研究空间并行规模和每条数据流的长度：

$$
P\in\{1,2,4,8,16\},
\qquad
M\in\{4,8,16,32,64,96\}.
$$

每个 $(P,M)$ 是一条独立的宽 QASM 和一个 Quafu task。对一个选定的
$P$：

1. 一次 `submit` 的 `Run All` 顺序创建六个 $M$ task；
2. 等六个任务执行完成；
3. 一次 `fetch` 的 `Run All` 逐个取回六个结果。

五个 $P$ 全部完成后共有：

$$
5\times6=30
$$

个 task ID。

这里的“一次 submit”是一次 Notebook 操作触发六次顺序 API 调用，不是
平台原子批处理。每个 task ID 返回后都会立即写入本地 registry；若中途
失败，已经提交的任务仍然有效。

## 为什么同时扫描 $P$ 和 $M$

- 增大 $M$：理想 QOS 通常更接近目标，但线路更深、硬件噪声更大；
- 增大 $P$：同一个 shot 内并行运行更多独立 stream，但使用更多物理
  qubits，也可能带来布局和共享噪声影响；
- 完整 $P\times M$ 网格能显示这两种效应是否存在交互。

每个 pair 先固定生成一条长度 96 的母 stream，较短 $M$ 只取前缀：

$$
s_{r,M}=s_{r,96}[0:M].
$$

同时较大的 $P$ 会保留较小 $P$ 的已有 streams。这样 $P$ 和 $M$ 两个
方向都形成嵌套、可匹配比较的设计。

## 推荐环境

在 VS Code 中打开 Notebook，选择 `QuantumComputing` kernel。

Notebook 不保存 API token。可以提前在终端设置：

```bash
export QPU_API_TOKEN='你的新 token'
```

不设置时，联网动作会显示隐藏输入框。

## 第一轮：$P=1$

配置单元保持：

```python
P_SELECTED = 1
ACTION = "dry_run"
CONFIRM_REAL_SUBMISSION = ""
CONFIRM_BATCH_DIGEST = ""
```

完整 `Run All`。它会离线生成并审计六条 QASM，并打印：

- `Batch digest`；
- 12 位 `Digest confirmation value`；
- `Fresh-batch confirmation`。

可选地设置：

```python
ACTION = "status"
```

完整 `Run All` 查看 Shenglian 状态和六个任务槽位。

真机提交时，把 dry-run 打印的值填入：

```python
ACTION = "submit"
CONFIRM_REAL_SUBMISSION = "SUBMIT_6_TASKS_FOR_P_1"
CONFIRM_BATCH_DIGEST = "这里粘贴 dry_run 打印的 12 位值"
```

完整 `Run All`。它会按 $M=4,8,16,32,64,96$ 顺序提交六个任务。
每个 task ID 会立即写入：

`results/qos_parallel_ensemble/task_registry.json`

提交后立刻清空两个确认字符串。等待六个任务全部结束，然后设置：

```python
ACTION = "fetch"
CONFIRM_REAL_SUBMISSION = ""
CONFIRM_BATCH_DIGEST = ""
```

完整 `Run All`。如果六个任务都已完成，一次 fetch 就能全部取回。如果
仍有任务排队，已完成部分会先保存；之后再次执行同一个 fetch 即可，
成功结果不会被 pending 状态覆盖。

## 后面四个 $P$

依次设置：

```python
P_SELECTED = 2
P_SELECTED = 4
P_SELECTED = 8
P_SELECTED = 16
```

每个 $P$ 重复一次六任务 submit、等待和一次批量 fetch。确认词相应改为：

```text
SUBMIT_6_TASKS_FOR_P_2
SUBMIT_6_TASKS_FOR_P_4
SUBMIT_6_TASKS_FOR_P_8
SUBMIT_6_TASKS_FOR_P_16
```

默认 ladder 会要求前一个 $P$ 的六个 $M$ 全部成功 fetch 且 returned
layout 审计通过，才允许提交更大的 $P$。

## 部分提交如何恢复

六次 API 调用不是原子的。如果提交在中间停止：

1. 查看 `task_registry.json`、`raw_results/receipt_*.json` 和
   `submission_intents/`；
2. 如果 intent 状态为 `submitting` 或 `ambiguous`，先在平台按其中的
   `task_name` 核实，不要直接重试：

   - 如果平台确实创建了任务，把映射填入
     `RECOVER_TASK_IDS_BY_M = {M: task_id}`；
   - 如果确认平台没有创建任务，把该 $M$ 放入
     `AMBIGUOUS_M_VERIFIED_NOT_SUBMITTED = (M,)`。

3. 设置：

   ```python
   RESUME_PARTIAL_BATCH = True
   ACTION = "submit"
   CONFIRM_REAL_SUBMISSION = (
       "RESUME_MISSING_TASKS_FOR_P_当前P"
   )
   CONFIRM_BATCH_DIGEST = "原批次的 12 位值"
   ```

Notebook 会识别 fingerprint，只补交缺失的 $M$，不会重复提交已有任务。

## 已有 task ID 的失败任务如何安全重试

关闭 Notebook 或重启 kernel 不会丢失真机任务。重新打开后需要
重新执行代码单元恢复变量，但 registry、receipt、raw result 和
分析表都从 `results/qos_parallel_ensemble/` 读取，不会要求重跑
已经成功的任务。

先运行一次 `ACTION="fetch"`，确保失败状态或 returned-layout
审计结果已落盘。对于当前 $P=2$ 的两个异常点，可先 dry-run：

```python
P_SELECTED = 2
RETRY_M_VALUES = (16, 96)
ACTION = "dry_run"
CONFIRM_REAL_SUBMISSION = ""
CONFIRM_BATCH_DIGEST = ""
```

核对输出后提交：

```python
ACTION = "retry"
CONFIRM_REAL_SUBMISSION = (
    "RETRY_FAILED_TASKS_FOR_P_2_M16_M96"
)
CONFIRM_BATCH_DIGEST = "P=2 原批次的 12 位值"
```

Notebook 会实时确认旧任务确实为终态失败，或者已有
`fetch_reason="audit_failed"`，然后只提交列出的 $M$。旧 task ID
会标为 `superseded` 并永久保留，新 task ID 成为该网格点的 active
记录。完成后设置：

```python
RETRY_M_VALUES = ()
RECOVER_RETRY_TASK_IDS_BY_M = {}
RETRY_M_VERIFIED_NOT_SUBMITTED = ()
CONFIRM_REAL_SUBMISSION = ""
CONFIRM_BATCH_DIGEST = ""
ACTION = "fetch"
```

如果 retry API 在返回 task ID 前中断，Notebook 会 fail-closed。
先按 retry intent 中的 `task_name` 到平台核实：已有远端任务时填
`RECOVER_RETRY_TASK_IDS_BY_M={M: task_id}`；确认没有时填
`RETRY_M_VERIFIED_NOT_SUBMITTED=(M,)`，再使用相同 retry 确认词和
digest 运行。

## 三维图和比较基线

主图的三个坐标是：

- $x=P$；
- $y=M$；
- $z=F_{\rm echo}$、硬件损失或 $P_{\rm eff}/P$。

其中 $P$ 和 $M$ 使用对数间距摆放，但 tick 标签仍显示真实数值。

第一幅三维面板同时比较：

1. hardware ensemble；
2. 完全相同固定 streams 的 noiseless ideal；
3. 解析的 $\mathbb E_{\rm IID}[F(M)]$。

硬件损失定义为：

$$
\Delta(P,M)
=
\mu_{\rm fixed}(P,M)-\widehat\mu_{\rm hw}(P,M).
$$

只有 30 个网格点全部取回并通过审计后，Notebook 才画完整 hardware
surface；缺失数据只显示为缺口或散点，不插值、不补零。

二维描述模型也只在严格审计达到 30/30 后拟合，并以正值表示硬件损失的
$\Delta$ 为响应；部分、不平衡的 fetch 网格不会提前生成容易误读的系数。

此外还会生成：

- 六个固定 $M$ 的二维 $P$ 切片和 cluster-aware 95% CI；
- 五个固定 $P$ 的二维 $M$ 切片，用来精确观察“理想改善 vs 线路加深”；
- hardware loss 热力图；
- $P_{\rm eff}/P$ 热力图；
- 有符号和绝对 same-shot pair correlation；
- returned two-qubit depth；
- returned/logical two-qubit gate ratio。

## 结果文件

所有结果位于：

`results/qos_parallel_ensemble/`

主要文件包括：

- `qasm/`：每个 $(P,M)$ 的宽 QASM；
- `manifests/`：streams、seeds、hash、理想值和资源；
- `batch_manifests/`：每个 $P$ 的六任务批次和 digest；
- `offline_audits/`：宽度、measurement 和跨-pair 门审计；
- `submission_intents/`：每次远程创建前后的提交意图；
- `raw_results/`：receipt、Quafu raw result、returned QASM；
- `task_registry.json`：全部 30 个 task ID；
- `hardware_tasks.csv/json`：每个 task 一行；
- `hardware_pairs.csv/json`：每个逻辑 pair 一行；
- `fetch_attempts.json`：幂等 fetch 尝试记录；
- `imdb_parallel_qos_grid_001_response_surface_3d.png`；
- `imdb_parallel_qos_grid_001_M_slices.png`；
- `imdb_parallel_qos_grid_001_P_slices.png`；
- `imdb_parallel_qos_grid_001_diagnostics_heatmaps.png`。

## 如何解释

一次 $(P,M)$ task 返回的是一个 $2P$-bit 联合分布。Notebook 对每个 pair
计算边缘 $P(00)$，不会使用随 $P$ 快速下降的全局 $P(0^{2P})$。

ensemble 误差条把一次 hardware shot 视为一个 cluster，因此保留 pair
之间的 same-shot 相关性。Notebook 保存：

- 未截断 $P_{\rm eff}$；
- $P_{\rm eff}/P$；
- design effect；
- signed pair correlation；
- absolute pair correlation。

负相关时可能出现 $P_{\rm eff}>P$，所以科学结果不把它强行截到 $P$。
这些量能提示共享噪声，但不能单独证明门串扰。

## 限制与停止条件

- 当前只确认 `[34,40]` 是已运行过的 Shenglian 连接边；
- $P>1$ 默认交给 qsteed 自动布局，returned QASM 会恢复实际物理 pairs；
- 同一 $P$ 的六个 $M$ 也可能映射到不同物理区域，Notebook 会报告
  layout consistency；
- 如果 $P=2$ 编译失败，不要继续扩大 $P$；
- $P=16,M=96$ 使用 32 个逻辑比特且线路很深，是压力测试，很可能接近
  噪声地板；
- `target_qubits` 只是允许使用的物理集合，不假设列表顺序是固定映射；
- returned-layout 审计失败的点会保留 raw 数据，但不进入三维主图；
- $1-F_{\rm echo}$ 与论文的 operator error $\epsilon$ 定义不同，不能
  直接当成同一纵轴；
- 每格只有一个 task，30 点响应面仍是 exploratory pilot；正式因果串扰
  或 mixed-effects 分析需要重复 campaign、交错任务顺序和物理布局轮换；
- 多个独立长度 $M$ 的 QOS 副本不等价于一个长度 $PM$ 的 coherent
  oracle。
