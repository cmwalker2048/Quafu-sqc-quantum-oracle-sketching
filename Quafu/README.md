# Quantum Oracle Sketching on Quafu-SQC

这是一个面向初学者、以可复现实验为目标的研究项目。项目研究论文
[Exponential quantum advantage in processing massive classical data](https://arxiv.org/abs/2604.07639)
中的 Quantum Oracle Sketching（QOS），核对其
[官方代码仓库](https://github.com/haimengzhao/quantum-oracle-sketching)，并把最基本的
Boolean phase oracle 重新写成可以检查、模拟和提交到 Quafu-SQC 云平台的门级实验。

项目已经从最小 Boolean 闭环推进到真实向量和真实数据：

1. 用 NumPy 验证 Boolean QOS 的数学机制；
2. 分开统计数据流随机性、量子 shots 和简化硬件噪声；
3. 生成可提交的 OpenQASM 2.0；
4. 通过 `quafusqc` 可选地提交到 Quafu-SQC 真机；
5. 用一个小型 centroid classifier 连接“量子态重叠”与分类读出；
6. 严格区分 expected operator 与真实 $\mathbb E[F_{\rm echo}]$；
7. 生成多控制、多规模、默认不提交的 Quafu-SQC campaign；
8. 逐数组复现 general-vector expected/active QOS 与 degree-4 arcsin-QSVT；
9. 在官方 IMDb 25,000 train / 25,000 test 上真实训练并评估 Ridge；
10. 用 general-vector QOS 重建 learned weight，再重新运行 held-out classification；
11. 生成 IMDb-derived flat QOS 与 full $D=4$ general-vector QSVT 真机包。

它仍不是完整量子 ridge/LS-SVM、matrix block encoding、amplitude
amplification 或 classical-shadows 复现，也不能用本地模拟和低维真机 pilot
证明论文的系统级渐近指数空间优势。

## 当前状态

| 模块 | 状态 | 准确含义 |
|---|---|---|
| Boolean QOS 数学模拟 | 已实现 | 显式采样、相位累计、`F_+` 验证 |
| 多数据流与 finite-shot 统计 | 已实现 | 区分 `R_streams`、`M_data`、`S_shots` |
| 简化硬件噪声模型 | 已实现 | 相位偏差、相位翻转、读出错误 |
| Quafu-SQC OpenQASM 2.0 生成 | 已实现 | 主 pilot 为 $n\le2$；$n=3$ 用于离线/sparse 压力测试 |
| Qiskit 线路可视化 | 已实现 | 解析同一 QASM，并在 notebook 内嵌线路图 |
| Expected-operator 对照 | 已实现 | 闭式、显式随机流与真实 fidelity 平均分开验证 |
| QASM/完整 statevector 等价 | 已实现 | $n=1,2,3$ 最大 $\ell_2$ 误差约 $9.16\times10^{-16}$ |
| Generic gate-level noise | 已实现 | Qiskit Aer；明确不是 Quafu-SQC 校准模型 |
| Quafu-SQC campaign bundle | 已实现 | 42 条 QASM、manifest、门预算与结果解析 |
| Quafu-SQC 云端提交 | 已接好，默认关闭 | 用户显式选择单条任务后开启 |
| Centroid/Hadamard-test 读出桥梁 | 已实现 | 使用理想态制备占位，不冒充完整 QOS 分类 |
| Flat $\pm1$ vector QOS | 已实现 | 数组闭式回归 + IMDb-derived 2-qubit QASM |
| 通用向量 QOS state sketch | 已实现 | expected/active 数组、heralding 与条件 fidelity 分开 |
| Full general-vector QSVT 门级 | 已实现 $D=4$ | 4 qubits、两辅助位、NumPy-vs-gate 逐振幅等价 |
| 官方 IMDb classification | 已实现 | 25k/25k 无泄漏 HashingVectorizer + Ridge |
| IMDb learned-weight QOS | 已实现 | active/expected 重建后重新做官方 test 分类 |
| tiny ridge + QSVT | 待实现 | 模拟器优先 |
| classical shadows | 待实现 | 在 Hadamard-test 验证之后 |

## 目录

```text
Quafu/
├── README.md
├── environment.yml
├── requirements.txt
├── .gitignore
├── assets/
│   ├── circuits/
│   │   ├── projector-phase-update.png
│   │   ├── general-vector-pauli-sample.png
│   │   ├── imdb-flat-qos-echo-overview.png
│   │   └── imdb-d4-general-qsvt-logical.png
│   └── figures/
│       ├── expected-unitary-validation.png
│       ├── general-vector-qos-scaling.png
│       ├── imdb_hashing_accuracy_and_formula.png
│       ├── imdb_general_vector_qos_sweep.png
│       └── imdb-d4-general-qsvt-scaling.png
├── docs/
│   ├── PROJECT_PLAN.md
│   ├── NEXT_STAGE_DECISION.md
│   ├── GENERAL_VECTOR_IMDB_RESULTS.md
│   ├── API_DECISION.md
│   ├── SOURCE_MAP.md
│   ├── VALIDATION.md
│   └── SECURITY.md
├── notebooks/
│   ├── 00_environment_and_quafu_access.ipynb
│   ├── 01_boolean_qos_core.ipynb
│   ├── 02_sampling_and_noise_benchmark.ipynb
│   ├── 03_quafu_boolean_qos_qasm.ipynb
│   ├── 04_centroid_readout_bridge.ipynb
│   ├── 05_boolean_qos_equivalence.ipynb
│   ├── 06_quafu_experiment_bundle.ipynb
│   ├── 07_general_vector_qos.ipynb
│   ├── 08_imdb_general_vector_classification.ipynb
│   ├── 09_quafu_imdb_vector_pilot.ipynb
│   └── 10_quafu_general_vector_qsvt_pilot.ipynb
├── data/
│   └── README.md
└── results/
    ├── README.md
    ├── general_vector_active_summary.csv
    ├── imdb_hashing_ridge_accuracy.csv
    ├── imdb_general_vector_qos_sweep.csv
    ├── imdb_vector_pilot/         # 65 条 flat QASM
    ├── general_vector_qsvt_pilot/ # 4-qubit full QSVT QASM
    └── quafu_bundle/              # 42 条 Boolean QASM
```

## 最快运行方式

### 已有本机环境

本机已经有一个可用的 Conda 环境：

```bash
conda activate QuantumComputing
cd ~/Desktop/Quafu
jupyter lab
```

如果 shell 中 `conda activate` 不工作，可以直接使用：

```bash
/opt/miniconda3/envs/QuantumComputing/bin/jupyter lab ~/Desktop/Quafu
```

请按编号依次运行 notebooks。`00`–`07` 默认离线；`08` 需要 IMDb 原始数据，
可设置 `IMDB_DATA_ROOT`，也可显式开启 notebook 内的官方下载 helper。`09`
复用同一数据，`10` 复用 `09` 保存的 $D=4$ target。`06`、`09`、`10` 都会
离线生成 QASM 和 manifest，所有云端提交开关默认是 `False`。真机时先审计
manifest，再只选择一个 experiment id。

### 新机器或 GitHub clone

```bash
git clone <你的仓库 URL>
cd Quafu
conda env create -f environment.yml
conda activate QuantumComputing
jupyter lab
```

若不使用 Conda，可在 Python 3.13 虚拟环境中执行
`python -m pip install -r requirements.txt`。真正提交前再单独配置平台凭据；
离线运行全部 notebook 不需要平台凭据。

## 为什么选择 quafusqc

本机 `QuantumComputing` 环境已经安装：

- Python 3.13.7
- `quafusqc 3.3.9`
- NumPy 2.3.3
- SciPy 1.16.1
- Matplotlib 3.10.6
- JupyterLab 4.6.0
- Qiskit 2.5.1（只负责线路解析、校验和可视化）
- Qiskit Aer 0.17.2（generic gate-level noise；不是 Quafu-SQC 校准模型）
- scikit-learn 1.9.0（官方 IMDb HashingVectorizer + Ridge）
- JAX/JAXLIB 0.8.1（固定原仓库回归）
- pyqsp 0.2.0（degree-4 arcsin-QSVT angle generation）

`quafusqc` 的安装名是 `quafusqc`，但代码中的导入名是：

```python
from quafu import Task
```

当前包的职责很窄：接收 OpenQASM 2.0、查询后端、提交任务和取回结果。
这种方式正适合第一阶段，因为生成的每一条门都可以直接审计。

QuarkStudio 当前没有安装。它可以在后续需要本地线路对象、后端拓扑展示或
更复杂编译流程时再引入；现在安装它不会改变 QOS 的核心数学，也不是完成
Boolean 实验的必要条件。详见 [工具链选型](docs/API_DECISION.md)。

## 核心实验

Boolean 目标 oracle 是

$$
O_f=\sum_x(-1)^{f(x)}|x\rangle\langle x|.
$$

对于均匀到来的经典样本 $(x_i,y_i)$，其中 $y_i=f(x_i)$，每个样本贡献

$$
V_i=\exp\left(i\frac{\pi N}{M}y_i|x_i\rangle\langle x_i|\right).
$$

同一地址被采样的次数接近 $M/N$，所以总相位接近

$$
\frac{\pi N}{M}\frac{M}{N}f(x)=\pi f(x),
$$

即理想的 $(-1)^{f(x)}$ 相位。

验证线路依次执行：

```text
|0...0> → H^n → QOS sketch → exact oracle inverse → H^n → measurement
```

![QOS 验证线路概览](assets/circuits/qos-verification-overview.png)

一个经典样本对应的 projector-phase 增量门如下。X mask 把指定地址暂时映射成
全 1，CCX 计算匹配标记，$P(\theta)$ 写入相位，随后反计算并撤销 mask：

![单样本 projector-phase 线路](assets/circuits/projector-phase-update.png)

扩展到 $n=3$ 时使用两个 clean work qubits，先计算前两位的 AND，再与第三位
合成 equality flag；写入相位后按相反顺序反计算：

![n=3 projector-phase 线路](assets/circuits/projector-phase-update-n3.png)

成功指标为：

$$
F_+=P(0^n), \qquad \epsilon_+=1-F_+.
$$

`F_+` 是针对均匀叠加输入的任务相关 fidelity，不是完整的 channel fidelity
或 diamond norm。

## General-vector 与 IMDb 实际结果

`07` 对固定 commit 的 expected/active general-vector QOS 做了逐数组回归。
在 $D=64$ 非平坦向量上，active 条件方向 fidelity 从约 0.465
（$M_{\rm fetched}=256$）升到约 0.979
（$M_{\rm fetched}=16384$）；对应 heralding probability 从约 7.31%
收敛到约 4.47%。block unitarity 误差小于 $2.5\times10^{-16}$。

![General-vector QOS scaling](assets/figures/general-vector-qos-scaling.png)

`08` 真正读取 official 25k train / 25k test。stateless signed hashing + Ridge
在 $D=256$ 的 held-out accuracy 为 67.744%，在 $D=16384$ 达到 85.196%。
在 $D=256$ 的最大 active QOS 预算上，20 条独立数据流得到条件方向 fidelity
$0.92796\pm0.01098$，重新预测全部 25,000 条 test 后的 accuracy 为
$0.67165\pm0.00213$；原 Ridge 为 0.67744。QOS logical-qubit 曲线仍被明确
标为 theory formula；分类 accuracy 是实际执行，不是由公式生成。

![IMDb measured accuracy and formula accounting](assets/figures/imdb_hashing_accuracy_and_formula.png)

下图把同一组 measured accuracy 映射到不同资源口径，用来说明空间研究动机；
其中 QOS logical-qubit 横轴是公式记账，不是真机测量。

![IMDb accuracy versus machine-size accounting](assets/figures/imdb_accuracy_vs_machine_size_accounting.png)

`09` 把 official-train 派生的 $D=4$ sign vector 编译成 65 条 flat-QOS QASM。
`10` 再实现完整两辅助位 general-vector QSVT；固定回归线路的 NumPy-vs-gate
后选择分支 $\ell_2$ 误差约 $2.62\times10^{-15}$。优化后的 $M=8$ verifier
仍有约 252 个本地 basis-level 二比特门，因此它是 deep proof-of-principle，
不是“4 qubits 就很容易”的演示。

![IMDb-derived full general-vector QSVT](assets/circuits/imdb-d4-general-qsvt-logical.png)

完整解释、结果表和“哪些是实测/仿真/公式”的边界见
[General-vector 与 IMDb 阶段结果](docs/GENERAL_VECTOR_IMDB_RESULTS.md)。

## 为什么不照抄 Figure 2

原 Figure 2 的任务性能来自经典 Ridge/SVD，量子部分是 logical-qubit
空间公式；IMDb 脚本还会先合并 official train/test、在全体文本上拟合
TF-IDF，再做交叉验证。照抄它不会得到无泄漏 official-test benchmark，也不会
验证 QOS 门级机制。`08` 因此改用 stateless hashing 和官方 split，同时保留
同一 logical-space 公式作为明确标注的理论记账。本项目已在
[原仓库审计与实现对应](docs/SOURCE_MAP.md)
完成关键审计，并把四面板复现延后为发布版附录。

这条 Boolean 验证链已经完成。当前优先级是：

```text
official IMDb split
        → normalized general-vector QOS
        → flat / full-QSVT QASM
        → Quafu-SQC counts + returned transpiled circuit
```

完整取舍、去重表与验收结果见
[下一阶段取舍](docs/NEXT_STAGE_DECISION.md)。

`05` 将 expected-operator surrogate、真实解析平均和 1,200 条显式随机流放在
同一张图中：

![Expected-unitary 验证](assets/figures/expected-unitary-validation.png)

`06` 再用实际门级线路说明：增加 $M$ 会降低理想采样误差，同时快速增加
二比特门压力；图中的噪声参数是 generic model，不是 Quafu-SQC 校准或真机结果。

![Hardware-aware QOS trade-off](assets/figures/hardware-aware-qos-tradeoff.png)

## 三种不要混淆的重复次数

| 符号 | 含义 |
|---|---|
| `M_data` | 一条 QOS 电路消耗的经典流式样本数 |
| `R_streams` | 独立生成多少条随机数据流/电路 |
| `S_shots` | 每条量子电路重复测量多少次 |

项目中的图和表应同时记录这三个量。不能把 shots 当成论文的数据样本复杂度。

## 云端运行边界

Quafu-SQC 的公开接口要求先构造完整 QASM，再整体上传。因此这里能验证：

```text
V_M ... V_2 V_1 的门级功能和真实硬件噪声
```

但不能严格证明整个经典控制器只用了 `poly(log N)` 空间。严格的“真 streaming”
还需要在同一段相干演化中实时接收样本、注入少量门、立即丢弃样本，而不是
预先保存完整线路。

因此真机实验应表述为：

> QOS gate-level functional demonstration on Quafu-SQC.

不应表述为：

> 已在当前云架构上实现完整系统级指数空间优势。

## 凭据安全

仓库内没有保存任何 API token。不要把 token 直接写入 notebook 单元、输出、
`.env` 或 Git commit。建议通过 `QPU_API_TOKEN` 环境变量或隐藏交互输入提供。

详见 [凭据安全](docs/SECURITY.md)。

## 下一阶段

离线 general-vector 与真实 IMDb 已闭合。下一条主线是：

1. 从 `09` manifest 选择一条 flat H-only / balanced control 上真机；
2. 保存 raw counts 与 returned transpiled QASM；
3. 若编译资源和 control 结果可接受，再尝试 `10` 的 $M=8$ full-QSVT verifier；
4. 分开报告 heralding probability、conditional fidelity 和成功事件数；
5. 再进入 matrix block encoding + tiny quantum ridge inverse；
6. Hadamard test 稳定后再做 classical shadows。

详细里程碑和验收条件见 [项目计划](docs/PROJECT_PLAN.md)。

本次本地执行结果和未验证项见 [本地验证记录](docs/VALIDATION.md)。

原仓库函数与本项目实现的逐项对应见
[原仓库审计与实现对应](docs/SOURCE_MAP.md)。

## 引用与许可

论文为 Zhao et al., arXiv:2604.07639 (2026)。正式发布到 GitHub 前，应根据
你的发布意图选择项目 LICENSE；当前阶段没有替你决定许可证。原论文代码仓库
使用 Apache-2.0，复用其代码时仍应保留对应版权和许可说明。

## 官方资料

- [Quafu-SQC 官方文档](https://quafu-sqc.readthedocs.io/en/latest/)
- [QuarkStudio 官方文档](https://quarkstudio.readthedocs.io/en/latest/)
- [Qiskit 线路可视化文档](https://quantum.cloud.ibm.com/docs/en/guides/visualize-circuits)
- [Quantum Oracle Sketching 代码仓库](https://github.com/haimengzhao/quantum-oracle-sketching)
