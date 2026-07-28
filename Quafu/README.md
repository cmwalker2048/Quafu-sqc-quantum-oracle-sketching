# Quantum Oracle Sketching on Quafu

这是一个面向初学者、以可复现实验为目标的研究项目。项目研究论文
**Exponential quantum advantage in processing massive classical data**
中的 Quantum Oracle Sketching（QOS），并把最基本的 Boolean phase oracle
重新写成可以检查、模拟和提交到 Quafu 云平台的门级实验。

当前项目刻意从最小闭环开始：

1. 用 NumPy 验证 Boolean QOS 的数学机制；
2. 分开统计数据流随机性、量子 shots 和简化硬件噪声；
3. 生成可提交的 OpenQASM 2.0；
4. 通过 `quafusqc` 可选地提交到 Quafu 真机；
5. 用一个小型 centroid classifier 连接“量子态重叠”与分类读出。

它还不是完整的 IMDb/PBMC、ridge regression、QSVT 或 classical shadows
复现，也不能用有限规模真机结果证明论文的渐近指数空间优势。

## 当前状态

| 模块 | 状态 | 准确含义 |
|---|---|---|
| Boolean QOS 数学模拟 | 已实现 | 显式采样、相位累计、`F_+` 验证 |
| 多数据流与 finite-shot 统计 | 已实现 | 区分 `K`、`M_data`、`S_shot` |
| 简化硬件噪声模型 | 已实现 | 相位偏差、相位翻转、读出错误 |
| Quafu OpenQASM 2.0 生成 | 已实现 | 第一版支持 1–2 个地址量子比特 |
| Qiskit 线路可视化 | 已实现 | 解析同一 QASM，并在 notebook 内嵌线路图 |
| Quafu 云端提交 | 已接好，默认关闭 | 需要新的有效 token，由用户手动开启 |
| Centroid/Hadamard-test 读出桥梁 | 已实现 | 使用理想态制备占位，不冒充完整 QOS 分类 |
| 通用向量 QOS state sketch | 待实现 | 下一阶段 |
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
│   └── circuits/
│       ├── bell-circuit.png
│       ├── projector-phase-update.png
│       ├── qos-verification-overview.png
│       ├── qos-full-n2.png
│       └── hadamard-overlap-test.png
├── docs/
│   ├── PROJECT_PLAN.md
│   ├── API_DECISION.md
│   ├── SOURCE_MAP.md
│   ├── VALIDATION.md
│   └── SECURITY.md
├── notebooks/
│   ├── 00_environment_and_quafu_access.ipynb
│   ├── 01_boolean_qos_core.ipynb
│   ├── 02_sampling_and_noise_benchmark.ipynb
│   ├── 03_quafu_boolean_qos_qasm.ipynb
│   └── 04_centroid_readout_bridge.ipynb
├── data/
│   └── README.md
└── results/
    └── README.md
```

## 最快运行方式

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

请按编号依次运行 notebooks。先运行 `00` 检查环境，但不要急着提交云任务；
`01` 和 `02` 完全离线；`03` 先生成和审计 QASM，确认后再把
`SUBMIT_TO_HARDWARE` 改为 `True`。

## 为什么选择 quafusqc

本机 `QuantumComputing` 环境已经安装：

- Python 3.13.7
- `quafusqc 3.3.9`
- NumPy 2.3.3
- SciPy 1.16.1
- Matplotlib 3.10.6
- JupyterLab 4.6.0
- Qiskit 2.5.1（只负责线路解析、校验和可视化）

`quafusqc` 的安装名是 `quafusqc`，但代码中的导入名是：

```python
from quafu import Task
```

当前包的职责很窄：接收 OpenQASM 2.0、查询后端、提交任务和取回结果。
这种方式正适合第一阶段，因为生成的每一条门都可以直接审计。

QuarkStudio 当前没有安装。它可以在后续需要本地线路对象、后端拓扑展示或
更复杂编译流程时再引入；现在安装它不会改变 QOS 的核心数学，也不是完成
Boolean 实验的必要条件。详见 `docs/API_DECISION.md`。

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

成功指标为：

$$
F_+=P(0^n), \qquad \epsilon_+=1-F_+.
$$

`F_+` 是针对均匀叠加输入的任务相关 fidelity，不是完整的 channel fidelity
或 diamond norm。

## 三种不要混淆的重复次数

| 符号 | 含义 |
|---|---|
| `M_data` | 一条 QOS 电路消耗的经典流式样本数 |
| `K_streams` | 独立生成多少条随机数据流/电路 |
| `S_shots` | 每条量子电路重复测量多少次 |

项目中的图和表应同时记录这三个量。不能把 shots 当成论文的数据样本复杂度。

## 云端运行边界

Quafu 的公开接口要求先构造完整 QASM，再整体上传。因此这里能验证：

```text
V_M ... V_2 V_1 的门级功能和真实硬件噪声
```

但不能严格证明整个经典控制器只用了 `poly(log N)` 空间。严格的“真 streaming”
还需要在同一段相干演化中实时接收样本、注入少量门、立即丢弃样本，而不是
预先保存完整线路。

因此真机实验应表述为：

> QOS gate-level functional demonstration on Quafu.

不应表述为：

> 已在当前云架构上实现完整系统级指数空间优势。

## 凭据安全

仓库内没有保存任何 API token。不要把 token 直接写入 notebook 单元、输出、
`.env` 或 Git commit。建议通过 `QPU_API_TOKEN` 环境变量或隐藏交互输入提供。

此前 demo 中出现过明文 token；凡是曾粘贴到聊天或 notebook 源码中的 token，
都应在平台侧撤销并重新生成。详见 `docs/SECURITY.md`。

## 下一阶段

在 Boolean QOS 的模拟、QASM 和真机小实验都稳定之后，再依次推进：

1. 通用实向量的 QOS state sketch；
2. $D=4$ 或 $8$ 的流式 centroid 分类；
3. tiny sparse ridge regression；
4. degree 3/5 的低阶 QSVT；
5. interferometric classical shadows；
6. 空间—精度—噪声相图及经典 fixed-memory baselines。

详细里程碑和验收条件见 `docs/PROJECT_PLAN.md`。

本次本地执行结果和未验证项见 `docs/VALIDATION.md`。

原仓库函数与本项目实现的逐项对应见 `docs/SOURCE_MAP.md`。

## 官方资料

- [Quafu SQC 官方文档](https://quafu-sqc.readthedocs.io/en/latest/)
- [QuarkStudio 官方文档](https://quarkstudio.readthedocs.io/en/latest/)
- [Qiskit 线路可视化文档](https://quantum.cloud.ibm.com/docs/en/guides/visualize-circuits)
- [Quantum Oracle Sketching 代码仓库](https://github.com/haimengzhao/quantum-oracle-sketching)
