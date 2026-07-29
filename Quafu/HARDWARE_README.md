# Quafu-SQC 真机验证操作指南

入口 notebook：
[`notebooks/11_quafu_hardware_validation.ipynb`](notebooks/11_quafu_hardware_validation.ipynb)

它已经把 QASM 审计、后端查询、提交、task registry、状态轮询、结果取回、
bit-order 规范化、returned circuit 统计和结果作图集中到一个文件。
本 notebook 使用本机已验证的 `quafusqc 3.3.9`：

```python
from quafu import Task
```

官方最新页面有时以 QuarkStudio 展示同一云端工作流；本实验不需要为此更换当前
已经通过离线验证的环境。任务字段与结果结构可对照
[Quafu-SQC 官方文档](https://quafu-sqc.readthedocs.io/en/latest/)。

## 1. 启动

```bash
conda activate QuantumComputing
cd ~/Desktop/Quafu
jupyter lab
```

打开 `notebooks/11_quafu_hardware_validation.ipynb`。如果 shell 中 Conda
命令不可用，可以直接运行：

```bash
/opt/miniconda3/envs/QuantumComputing/bin/jupyter lab ~/Desktop/Quafu
```

`09` 和 `10` 生成的 QASM、manifest 已包含在项目中，因此真机 notebook
不需要重新下载 IMDb，也不需要重新训练模型。

## 2. 顶部配置区

最重要的字段是：

```python
ACTION = "dry_run"
BACKEND = "Baihua"
BITSTRING_ORDER = "qiskit"
CONFIRM_REAL_SUBMISSION = ""
MAX_TASKS_PER_SUBMISSION_CELL = 2
ALLOW_DEEP_QSVT = False
```

`ACTION` 可选：

| 值 | 行为 |
|---|---|
| `dry_run` | 只审计本地线路；不读取凭据、不联网、不提交 |
| `status` | 只查询后端队列 |
| `submit` | 提交 enabled 任务，保存 task id 后立即结束 |
| `submit_and_wait` | 提交、轮询，并在完成后取回 |
| `fetch` | 不提交；从 registry 或手工 task id 取回结果 |

建议先把整本 notebook 以 `dry_run` 完整运行一次。正确输出应包含：

```text
Offline QASM audit PASS: 73 circuits; failures: 0
Counts and returned-QASM parser self-tests PASS.
Dry run: no token read, no network request.
```

然后使用：

```python
ACTION = "status"
```

查看当前可用后端和队列。非负整数表示排队任务数；`Offline` 或
`Maintenance` 不可提交。

## 3. 提交方式

真正提交时：

```python
ACTION = "submit"
CONFIRM_REAL_SUBMISSION = "RUN_QUAFU_HARDWARE"
```

在 `RUN_PLAN` 中只把当前阶段需要的条目设为：

```python
"enabled": True
```

其他条目保持 `False`。默认一次最多提交两条。每次成功返回 task id 后，
notebook 立即保存 receipt 和 `task_registry.json`。如果网络返回不明确，
不要直接再次提交；先在 registry 和平台任务列表中确认，避免重复任务。

默认编译选项：

```python
DEFAULT_COMPILER = "quarkcircuit"
USE_READOUT_CORRECTION = False
OPEN_DD = None
TARGET_QUBITS = []
```

第一轮保持 correction 和 dynamical decoupling 关闭，避免同时改变多个变量。
`TARGET_QUBITS=[]` 让远端编译器选择布局。若手工指定，数量必须与当前线路的
2 或 4 qubits 完全一致，且不能重复。

## 4. 必须按顺序执行

### 阶段 A：2-qubit bit order

默认已经启用：

```text
cal_bitorder_2q_x_q0
cal_bitorder_2q_x_q1
```

提交并取回后，在汇总表检查：

| 线路 | Qiskit 约定的主峰 |
|---|---|
| `X(q0)` | `01` |
| `X(q1)` | `10` |

如果返回主峰正好相反，把顶部配置改为：

```python
BITSTRING_ORDER = "reversed"
```

然后用原 task id 重新执行 `fetch`。这只重新解析已经取得的数据，不要重新提交。
表中 `calibration_dominant_matches_expected` 应为 `True`。

### 阶段 B：4-qubit bit order

关闭两条 2q 校准，开启：

```text
cal_bitorder_4q_x_q0
cal_bitorder_4q_x_q3
```

Qiskit 约定下主峰应为：

| 线路 | 主峰 |
|---|---|
| `X(q0)` | `0001` |
| `X(q3)` | `1000` |

full QSVT 的 classical string 定义为：

```text
c3 c2 c1 c0 = real signal address[1] address[0]
```

所以规范化后：

```text
herald success: bits[:2] == "00"
joint target success: bits == "0000"
```

在两条 4q 校准都通过前，notebook 默认禁止提交 full QSVT。

### 阶段 C：flat controls

依次运行，不要一次全部开启：

1. `imdb_flat_001_h_only_M0_r0`
2. `imdb_flat_005_synthetic_entangled_balanced_M8_r0`
3. `imdb_flat_002_exact_pair_M0_r0`
4. `imdb_flat_003_balanced_stream_M16_r0`
5. `imdb_flat_004_phase_count_probe_M16_r0`

前三个主要用于分离读出、纠缠门和 echo 链误差。H-only、exact-pair 和
balanced-stream 的理想 $F_{\rm echo}$ 都为 1。

phase-count probe 的理想输出分布为：

| bitstring | probability |
|---|---:|
| `00` | 0.728553 |
| `01` | 0.021447 |
| `10` | 0.125000 |
| `11` | 0.125000 |

### 阶段 D：编译器 A/B

对同一条 balanced control 分别开启：

```text
control_balanced_qc
control_balanced_qsteed
```

两条任务应尽量在相近时间运行。比较：

- raw $F_{\rm echo}$；
- returned two-qubit gates；
- returned two-qubit depth；
- SWAP；
- returned QASM 是否仍包含三比特门。

选择结果更稳定、路由代价更低的编译器后，再固定后续 campaign。

### 阶段 E：少量 IMDb IID

预置了三个接近各自规模总体表现的独立流：

```text
imdb_flat_007_iid_M4_r1
imdb_flat_033_iid_M16_r7
imdb_flat_056_iid_M32_r10
```

先各运行一条，观察随 $M$ 增加的理想采样收益是否被门噪声抵消。不要直接提交
全部 60 条 IID。若扩展 campaign，建议按 replicate 交错 M4/M16/M32，并周期性
插入 H-only 来监测设备漂移。

### 阶段 F：full general-vector QSVT

只有前面 controls 和 4q bit order 合格后才设置：

```python
ALLOW_DEEP_QSVT = True
```

先成对开启：

```text
general_qsvt_m8_prepare
general_qsvt_m8_verify
```

当前可提交 M8 QASM 的理想值为：

| 指标 | 理想值 |
|---|---:|
| heralding probability | 0.051243 |
| verifier conditional fidelity | 0.463279 |
| 本地编译 prepare/verify 二比特门 | 251 / 252 |

`prepare` 用于测 heralding；只有 `verify` 含 target inverse，才能从全零事件
得到 conditional fidelity。两条任务应在同一设备时间窗运行。

M8 有可辨认的 heralding 和方向信号后，再考虑：

```text
general_qsvt_m16_prepare
general_qsvt_m16_verify
```

M16 理想 $p_{\rm succ}=0.032233$、$F_{\rm cond}=0.626404$，但本地编译后二比特门
已达 443/444，硬件风险明显更高。

## 5. 取回结果

提交后可以关闭 notebook。稍后重新打开，在配置区设置：

```python
ACTION = "fetch"
FETCH_RUN_LABELS = ["bitorder_2q_xq0", "bitorder_2q_xq1"]
```

若要取回 registry 中全部任务：

```python
FETCH_RUN_LABELS = ["*"]
```

如果 task id 来自其他 session，则使用：

```python
MANUAL_FETCH_PLAN = [
    {
        "run_label": "smoke_h_only_qc",
        "experiment_id": "imdb_flat_001_h_only_M0_r0",
        "compiler": "quarkcircuit",
        "task_id": 123456789,
    },
]
```

未完成任务只记录状态，不会自动重新提交。`Task.result()` 返回后，notebook
仍会检查 payload 的 `status` 和 `count`，避免把非终态字典当作实验结果。

## 6. 输出文件

```text
results/hardware_validation/
├── offline_audit.json
├── offline_audit.csv
├── audited_run_plan.csv
├── calibration_qasm/
├── task_registry.json
├── hardware_results.json
├── hardware_results.csv
└── raw_results/
    ├── backend_status_*.json
    ├── receipt_*.json
    ├── result_*.json
    ├── returned_*.qasm
    └── hardware_validation_results.png
```

`raw_results/`、registry 和汇总的真实设备结果默认被 `.gitignore` 排除。
准备发布时先人工检查设备、任务和编译信息，再决定哪些文件应进入 Git。

raw JSON 会保留平台返回的：

- `count`；
- `corrected`；
- `transpiled`；
- `status`；
- `tid`；
- `finished`；
- `error`；
- `qlisp`。

## 7. 如何判断结果

每条完成任务首先检查：

- `result_status` 为 Finished/Completed/Success；
- `result_error` 为空；
- `shots_match_manifest=True`；
- `task_id_matches=True`；
- `returned_qasm_parse_reason="ok"`；
- bit-order calibration 的 dominant bitstring 正确。

flat：

$$
\widehat F_{\rm echo}=\frac{N_{00}}{N}.
$$

full QSVT verifier：

$$
\widehat p_{\rm succ}
=\frac{N_{\mathrm{real}=0,\mathrm{signal}=0}}{N},
$$

$$
\widehat F_{\rm cond}
=\frac{N_{0000}}
{N_{\mathrm{real}=0,\mathrm{signal}=0}}.
$$

notebook 会同时给出 95% Wilson 区间。$F_{\rm cond}$ 的分母是 heralding
success events，不是总 shots。对于 $D=4$，无方向信息的均匀 conditional
baseline 是 $1/4$；`directional_signal_ci_above_uniform_1_over_4=True`
表示 95% 区间下界仍高于该 baseline。

不能只看 $F_{\rm cond}$：

- 必须同时报告 $p_{\rm succ}$ 和 success events；
- 噪声把 ancilla 随机化时，$p_{\rm succ}$ 可能向 $1/4$ 漂移，这不是改进；
- full QSVT 必须和相邻时间运行的 prepare/control 一起解释。

## 8. 常见问题

### Duplicate submission blocked

同一 label、线路和编译选项已经存在 task id。先 `fetch`；若确实要做独立重复，
换一个新的 `run_label`。只有明确需要时才设置
`ALLOW_DUPLICATE_SUBMISSION=True`。

### Full QSVT blocked

需要同时满足：

- `ALLOW_DEEP_QSVT=True`；
- 两条 4q bit-order 校准已经完成；
- 当前 `BITSTRING_ORDER` 下二者 dominant bitstring 均正确。

### 任务仍是 Submitted

这是正常排队状态。保留 task id，稍后再 `fetch`。不要因 notebook 轮询超时
而再次提交。

### returned QASM 无法解析

原始 `transpiled` 文本和 hash 仍会保存。把
`returned_qasm_parse_reason`、raw JSON 和 returned QASM 一起保留，不能把
资源字段的 `NaN` 当成零门。

### corrected counts 为空

默认 `USE_READOUT_CORRECTION=False`，因此正常。若后续单独开启 correction，
raw 和 corrected 必须分列报告；corrected 可能是非整数权重。

## 9. 结论边界

这一套实验能回答：

- QOS/QSVT 门级机制能否在 Quafu-SQC 真机编译和执行；
- 随线路深度增加，echo、heralding 和条件方向信息如何衰减；
- 不同编译器和路由带来的二比特门代价。

它不能单独证明官方 IMDb 25k/25k 任务已经在量子硬件上端到端运行，也不能用
$D=4$ pilot 宣称系统级指数空间优势。真实 IMDb accuracy 结果仍来自 `08`；
真机 notebook 验证的是对应 state-sketch primitive 的设备可执行性。

