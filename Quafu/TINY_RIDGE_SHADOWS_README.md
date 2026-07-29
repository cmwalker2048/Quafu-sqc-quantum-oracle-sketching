# Tiny Ridge + QSVT + Classical Shadows 真机指南

入口 notebook：
[`notebooks/12_tiny_ridge_qsvt_shadows.ipynb`](notebooks/12_tiny_ridge_qsvt_shadows.ipynb)

这一本把以下链路放在同一个可复现实验中：

$$
\text{official IMDb train}
\rightarrow
\text{2D Ridge}
\rightarrow
\text{inverse QSVT}
\rightarrow
\text{interferometric global-Clifford shadow}
\rightarrow
\text{official IMDb test accuracy}.
$$

经典训练、特征投影和最终分类在本地运行；只有 4-qubit QSVT、校准线路及
shadow 测量线路提交到 Quafu-SQC。

## 1. 这次真正实现了什么

- 只用 official train 选择词汇和训练 2D Ridge，official test 只作最终评估；
- 为 $2\times2$ Ridge normal-equation matrix 构造 block encoding；
- 用 degree-3、spectrum-matched 奇多项式执行 inverse QSVT；
- postselect `real=0, signal=0` 后得到 Ridge weight direction；
- 使用干涉态

$$
|\widetilde w\rangle
=\frac{|0\rangle|0\rangle+|1\rangle|w\rangle}{\sqrt 2}
$$

  保留 $\operatorname{Re}\langle x|w\rangle$ 的正负号；
- 用 12 个可复现的 two-qubit global Clifford settings 做 batched
  classical-shadow 重建；
- 将重建出的两个权重重新用于全部 25,000 条 official test 评论；
- 保存 QASM、hash、task receipt、counts、returned QASM、CX/depth 和最终图。

本地已验证的主要数值：

| 项目 | 本地理想结果 |
|---|---:|
| tiny Ridge official-test accuracy | 0.84848 |
| trivial polarity rule accuracy | 0.84876 |
| normal-equation matrix condition number | 2.20413 |
| 对应 Ridge singular-value $\kappa_{\rm reg}$ | 1.48463 |
| inverse-QSVT heralding probability | 0.10457 |
| postselected weight-state fidelity | 约 1.0 |
| interferometric circuit heralding probability | 0.18934 |
| 12-setting Aer QSVT-shadow accuracy | 约 0.84900 |
| 12-setting QSVT-shadow weight norm ratio | 约 0.623 |
| 12-setting QSVT-shadow signed-overlap RMSE | 约 0.303 |
| Ridge prepare/verify 本地编译 CX | 18 / 18 |
| full QSVT-shadow 本地编译 CX | 约 95–97 |

最后一行说明 full shadow 是高风险深线路。12 个 settings 已能恢复分类符号，
但 overlap 幅值仍有明显误差，所以它是 sign/classification pilot，不是论文
Lemma F.16 的 $\epsilon$-accurate overlap 保证。先运行浅层 control，再决定
是否花费真机额度运行 full QSVT-shadow。

## 2. 科学边界

论文 Figure 2 的三条曲线共享经典 `RidgeClassifier` accuracy，只改变机器空间
的记账公式。原始真实数据脚本没有执行 QOS、QSVT 或 classical shadows。

本 notebook 新增的是一个真实可执行的 4-qubit **tiny mechanism bridge**，
但它不计入监督式词汇投影的经典存储，也没有实现 scalable QOS data loading。
因此：

- 可以称为 tiny Ridge inverse-QSVT + shadow 的真机机制验证；
- 不能称为复现 Figure 2 的端到端量子优势；
- 不能用 4-qubit 线路证明 IMDb 的指数空间优势。

因此最终图把 paper-inspired machine-size account 和 tiny experiment 放在两个
独立面板，不再把 4 个物理 qubits 画到整机空间账户的同一纵轴。

## 3. 在 VS Code 或 Jupyter 中启动

在 VS Code 中安装 Jupyter 扩展，打开项目文件夹并选择
`QuantumComputing` kernel。也可以在终端启动：

```bash
conda activate QuantumComputing
cd ~/Desktop/Quafu
jupyter lab
```

如果 `conda activate` 不可用：

```bash
/opt/miniconda3/envs/QuantumComputing/bin/jupyter lab ~/Desktop/Quafu
```

首次打开后，先确认右上角 kernel 是 `Python (QuantumComputing)`。

## 4. 第一次只做离线验证

保持配置：

```python
ACTION = "dry_run"
RUN_STAGE = "calibration"
CONFIRM_REAL_SUBMISSION = ""
```

然后 `Run All`。这一步不读取 token、不联网、不提交任务。正确结果应包含：

```text
Generated QASM: 31 entries
Offline QASM audit PASS: 31 circuits; failures: 0
Dry run: no token read, no network request.
```

它会生成 31 条 QASM：

| 阶段 | 数量 | 作用 |
|---|---:|---|
| `calibration` | 4 | q0–q3 完整 4q bit-order |
| `ridge` | 3 | direct control、QSVT prepare、QSVT verify |
| `shadow_control` | 12 | 浅层直接态制备 + shadows |
| `shadow_qsvt` | 12 | 完整 QSVT 输出 + shadows |

## 5. Token 和后端状态

不需要把 API token 写入 notebook。设置网络动作后，代码会：

1. 优先读取当前进程的 `QPU_API_TOKEN`；
2. 如果没有，则显示隐藏输入框，让你临时粘贴 token。

只查状态时：

```python
ACTION = "status"
```

然后 `Run All`。这会完整重建并审计本地线路，查询后端队列，并逐个查询
registry 中已有 task id 的状态，不提交。`Baihua` 对应非负整数时，该整数是
当前队列长度；`Offline` 或 `Maintenance` 时不要提交。

## 6. 真机运行顺序

每次提交都需要：

```python
ACTION = "submit"
CONFIRM_REAL_SUBMISSION = "RUN_TINY_RIDGE_QSVT"
```

一次最多提交两条，成功后立即把 task id 写入 registry。同一个
run label、backend、compiler 和选项的重复提交默认会被拦截。

保持 `USE_READOUT_CORRECTION=False`。当前统计使用原始非负整数 counts 和二项
Wilson 区间；notebook 会主动拒绝把 corrected quasi-counts 当成普通 shots。

如果目标是完成整条 shadow 链，请在第一次 calibration 前，从平台拓扑中选一组
连通的四个物理 qubits，并在所有阶段保持一致，例如：

```python
TARGET_QUBITS = [q_a, q_b, q_c, q_d]
```

必须把占位符替换成当前 backend 的真实物理编号。full shadow 默认拒绝自动布局；
它还要求 calibration、Ridge 和 shadow control 与当前 backend、compiler、
layout 属于同一 cohort。

### A. 4q bit-order 校准

```python
RUN_STAGE = "calibration"
BATCH_START = 0
BATCH_SIZE = 2
```

这是第一批 q0/q1。第二批使用：

```python
BATCH_START = 2
BATCH_SIZE = 2
```

Qiskit bit order 下四条预期主峰：

| 线路 | 主峰 |
|---|---|
| `tiny_cal_4q_x_q0` | `0001` |
| `tiny_cal_4q_x_q1` | `0010` |
| `tiny_cal_4q_x_q2` | `0100` |
| `tiny_cal_4q_x_q3` | `1000` |

如果四者整体反向，取回和分析前设置：

```python
BITSTRING_ORDER = "reversed"
```

改变 bit order 后只需重新 `fetch`，不应重新提交。

### B. Ridge QSVT

此阶段有三条任务，所以分两批：

第一批：

```python
RUN_STAGE = "ridge"
BATCH_START = 0
BATCH_SIZE = 2
```

第二批：

```python
RUN_STAGE = "ridge"
BATCH_START = 2
BATCH_SIZE = 1
```

`tiny_ridge_qsvt_prepare` 只测 heralding 和条件 system population，不把
system-zero 概率误称为 fidelity。只有 `tiny_ridge_qsvt_verify` 含
target inverse，可以估计 postselected conditional fidelity。notebook 同时给
二项 Wilson 95% 区间。

### C. 浅层 shadow control

```python
RUN_STAGE = "shadow_control"
BATCH_SIZE = 2
```

依次把 `BATCH_START` 设为：

```text
0, 2, 4, 6, 8, 10
```

12 个 setting 都取回后再把它当作完整 control 结果。只有 2–11 个 setting 时，
notebook 仍会显示临时估计，但图例会明确标出已取得的 setting 数量。

### D. 完整 QSVT-shadow

只有 calibration、Ridge verifier 和 shadow control 都有可辨认信号后才开启：

```python
RUN_STAGE = "shadow_qsvt"
ALLOW_FULL_QSVT_SHADOW_SUBMISSION = True
BATCH_SIZE = 2
HARDWARE_COHORT_TASK_IDS = [
    # 从 task_registry.json 复制：
    # 4 条 calibration + 1 条 tiny_ridge_qsvt_verify
    # + 12 条 shadow_control，共 17 个互不重复的正整数 task id
]
CONFIRM_COHERENT_HARDWARE_COHORT = "I_REVIEWED_TASK_TIME_WINDOW"
```

先在平台核对这 17 条任务的提交/执行时间，确认它们属于你愿意比较的同一设备
时间窗，并且 backend、compiler、四个 physical qubits 完全一致；确认字符串
表示你已经完成这次人工核对，不是自动判断设备漂移。

代码会要求每个 prerequisite experiment 恰好有一条可信结果，并检查：

| 发布门槛 | 默认阈值 |
|---|---:|
| 每条 calibration 的 expected-state Wilson 95% 下界 | $\ge 0.50$ |
| Ridge verifier heralding Wilson 95% 下界 | $\ge 0.02$ |
| Ridge conditional fidelity Wilson 95% 下界 | $>0.50$ |
| 每条 shallow control heralding Wilson 95% 下界 | $\ge 0.75$ |
| shallow signed-cosine setting-bootstrap 95% 下界 | $\ge 0.50$ |
| shallow signed-overlap RMSE | $\le 0.50$ |
| shallow accuracy setting-bootstrap 95% 下界 | $\ge 0.70$ |

这些是防止失败信号被误放行的保守工程门槛，不是论文的复杂度定理，也不是
i.i.d.-Clifford median-of-means 置信保证。任何一项失败都会阻止 full-QSVT
提交；仅打开布尔开关不能绕过。

同样依次使用 `BATCH_START = 0, 2, 4, 6, 8, 10`。这些线路本地约有
95–97 个 CX，远端路由后可能更多，因此前两条应视为 hardware feasibility
probe。returned QASM 的 CX 和 depth 很高、或 heralding 完全不可辨认时，可以
停止后续批次；不要为了凑齐 12 条而继续消耗额度。

## 7. 如何等待和取回

提交后可以关闭 notebook。任务完成时间取决于实时队列。稍后重新打开：

```python
ACTION = "fetch"
FETCH_RUN_LABELS = ["*"]
```

然后 `Run All`。代码会逐个执行 `status(task_id)`：

- 未完成：只保存状态，不调用 result endpoint；
- 已完成：取回 counts、原始 payload 和 returned QASM；
- 不会重新提交，也不需要再次改成 `submit`。

也可以只取某几条：

```python
FETCH_RUN_LABELS = [
    "tiny_ridge_qsvt_prepare",
    "tiny_ridge_qsvt_verify",
]
```

如果 task id 不是当前 notebook 提交的，可在 `MANUAL_FETCH_PLAN` 中填写
`run_label`、`experiment_id`、`compiler`、正整数 `task_id`，以及原提交时的
`qasm_sha256`、`model_sha256`、`data_summary_sha256`。缺少 provenance hash
的手工任务仍会保存 raw payload，但会以 `provenance_mismatch` 拒绝分析。

notebook 会把每次提交时的完整 manifest entry（包括 Clifford seed/matrix）和
模型哈希冻结进 receipt。重新生成 QASM 后取回旧任务时，只要 hash 不一致，就
不会拿当前 Clifford 错误解码旧 counts。

如果同一 experiment 有多次可信运行，汇总会要求显式设置
`HARDWARE_COHORT_TASK_IDS`。提交 full 阶段前，它至少包含选定的 17 条前置
任务；12 条 full-QSVT 结果取回后，再把这 12 个 task id 加入，形成 29 条
最终 cohort。这样图中只会使用你明确选定且同 backend/compiler/layout 的
calibration、verifier、control 和 full-shadow 结果，防止静默混合不同批次。

## 8. 结果应怎样读

证据链按下面顺序判断：

1. 四条 one-hot calibration 的主峰符合 bit-order 假设；
2. Ridge QSVT 的 heralding 概率与理想 0.10457 有可辨认的一致性；
3. verifier 在 herald 条件下仍保有 weight-state signal；
4. shadow control 能恢复正确权重符号和分类方向；
5. full QSVT-shadow 在相同 Clifford seeds 下仍能恢复方向；
6. 同时报告 accepted events、setting 数、returned CX/depth 和探索性区间。

简单的 $w\propto(1,-1)$ polarity rule accuracy 是 0.84876，略高于 exact
tiny Ridge 的 0.84848。因此分类 accuracy 不是 QSVT solver 增益证据；solver
证据来自 heralding 和 target-inverse verifier。

12 个 settings 很少，因此 setting-level bootstrap 区间可能很宽，且它只是固定
test set 上的探索性 cluster bootstrap，不是论文 i.i.d.-Clifford +
median-of-means 的严格置信保证。重复 shots 降低每个 setting 内的测量噪声，
但不能冒充更多独立随机 Cliffords。正式扩展实验时应增加
`N_SHADOW_SETTINGS`，并重新生成、审计和提交相应 QASM。

## 9. 输出文件

```text
results/tiny_ridge_qsvt_shadows/
├── tiny_imdb_projection.npz
├── tiny_imdb_projection.json
├── tiny_ridge_qsvt_model.json
├── manifest.json
├── manifest.csv
├── offline_qasm_audit.csv
├── local_ideal_counts.json
├── local_ideal_summary.json
├── qasm/
├── task_registry.json
├── hardware_results.json
├── hardware_results.csv
└── raw_results/
```

真机 receipt、raw payload、returned QASM 和 task registry 默认被 Git 忽略。
先检查其中是否有账户或设备敏感字段，再选择性发布。

最终图：

- `assets/figures/tiny-ridge-machine-size-comparison.png`
- `assets/figures/tiny-ridge-qsvt-polynomial.png`
- `assets/figures/tiny-ridge-shadow-accuracy.png`

真机 full-shadow counts 被取回后，同一次 `Run All` 会在 fetch 之后重新读取
结果并更新图。要标成 `final`，`HARDWARE_COHORT_TASK_IDS` 必须包含完整的
29 条最终 cohort，人工时间窗确认必须存在，而且所有前置统计门槛均通过。
少于全部 settings、重复 setting、没有固定 layout 或前置质量不足时，只作为
`provisional`/withheld 结果，不生成正式硬件点。
