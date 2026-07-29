# General-vector QOS 与真实 IMDb 阶段结果

验证日期：2026-07-29  
论文：[Exponential quantum advantage in processing massive classical data](https://arxiv.org/abs/2604.07639)  
固定源码：[haimengzhao/quantum-oracle-sketching](https://github.com/haimengzhao/quantum-oracle-sketching)，commit `10c092cefcfdff9951bf5729bd2ffb4c25fe2254`

## 先给结论

`05` 和 `06` 的主要职责确实是验证 Boolean QOS 的数学、QASM 和实验控制链。
真正进入 general-vector 与真实任务的是 `07`–`10`：

| Notebook | 真正完成的事情 | 证据层级 |
|---|---|---|
| `07_general_vector_qos` | 对任意实向量运行 expected/active QOS，统计 heralding、条件 fidelity、样本规模与 QSVT 近似 | 固定源码回归 + NumPy 数组仿真 |
| `08_imdb_general_vector_classification` | 在 Stanford IMDb 官方 25k train / 25k test 上训练 Ridge；用 QOS 重建 learned-weight direction 后重新预测全部 test | 真实数据、真实 held-out 分类 + active QOS 数组仿真 |
| `09_quafu_imdb_vector_pilot` | 从 official-train 学到 $D=4$ 向量，生成 65 条低深度 flat-QOS 真机线路 | IMDb-derived 数据 + 可提交 QASM + 理想 statevector |
| `10_quafu_general_vector_qsvt_pilot` | 将同一 IMDb 向量编译为完整 degree-4 general-vector QSVT、real-part LCU 和 heralding 线路 | 4-qubit 完整门级 QASM + Aer shots + NumPy 逐振幅等价 |

因此，项目现在已经有真实 IMDb 分类结果和真实可提交线路；尚缺的是
Quafu-SQC 设备返回的 counts、校准信息和远端 transpiled circuit。没有这些返回前，
不能声称已经在真机上测得量子优势。

## 1. General-vector QOS：实际数值结果

`07` 使用 $D=64$ 的非平坦实向量，而不是 Boolean 或仅有 $\pm1$ 的特殊情形。
active QOS 每个预算使用 40 条独立数据流，并用 2,000 次 stream bootstrap
给出区间。

| $M_{\rm fetched}$ | $M_{\rm used}$ | mean conditional fidelity | mean heralding probability |
|---:|---:|---:|---:|
| 256 | 192 | 0.46498 | 0.07312 |
| 1,024 | 768 | 0.77970 | 0.05463 |
| 4,096 | 3,072 | 0.92641 | 0.04775 |
| 16,384 | 12,288 | 0.97905 | 0.04471 |

这个结果说明：数据流变长后，herald 成功分支的方向确实收敛到目标向量。
同时，成功概率约为 4.5%，所以条件 fidelity 不能脱离 heralding 成本单独展示。

固定源码回归还得到：

- expected NumPy 与固定 JAX 源码最大误差：$2.05\times10^{-15}$；
- active NumPy 与固定 JAX 源码最大误差：$9.19\times10^{-17}$；
- signal block 最大 unitarity 误差：$2.50\times10^{-16}$；
- degree-4 流程只使用 fetched samples 的 75%，其余 25% 已单独记账。

原始表在
[`results/general_vector_active_summary.csv`](../results/general_vector_active_summary.csv)，
缩放图在
[`assets/figures/general-vector-qos-scaling.png`](../assets/figures/general-vector-qos-scaling.png)。

## 2. 官方 IMDb：真实 held-out classification

数据使用 Stanford ACL IMDb 的官方 split：

- train：25,000 条，positive/negative 各 12,500；
- test：25,000 条，positive/negative 各 12,500；
- `train/unsup` 排除；
- HashingVectorizer 无需在 test 上拟合词表；
- Ridge 只在 official train 上拟合，test 只用于最终预测。

classical baseline 的实测 official-test accuracy 为：

| hashing dimension $D$ | 流式特征所需 qubits $\lceil\log_2D\rceil$ | measured test accuracy |
|---:|---:|---:|
| 256 | 8 | 0.67744 |
| 1,024 | 10 | 0.76132 |
| 4,096 | 12 | 0.82000 |
| 16,384 | 14 | 0.85196 |

这里的 qubit 列只是“保存一个 hashed feature index”的流式地址宽度，不是完整
QOS/Ridge 算法所需的总逻辑 qubits。

为了让 general-vector QOS 与可承受的实验规模连接，主实验选择 $D=256$ 的
Ridge learned weight，先归一化成

$$
u=\frac{w}{\|w\|_2},
$$

再用未修改的 active QOS source core 重建方向。重建后乘回 $\|w\|_2$、保留
原 Ridge intercept，并重新对 25,000 条 official-test 影评作预测。

最大预算每次取得 16,384 个样本，其中 QSVT 实际使用 12,288 个；20 条独立流的
结果为：

| 方法 | success probability | conditional fidelity | official-test accuracy |
|---|---:|---:|---:|
| 原始 $D=256$ Ridge | — | 1.00000 | 0.67744 |
| expected QOS | 0.01566 | 0.99990 | 0.67752 |
| active QOS，mean $\pm$ std | $0.04769\pm0.00190$ | $0.92796\pm0.01098$ | $0.67165\pm0.00213$ |

因此，在这个可复现实验点上，active sketch 保留了约 99.15% 的 baseline
classification accuracy：

$$
\frac{0.671652}{0.67744}\approx 0.9915.
$$

这是真实 held-out prediction 的结果；量子子程序本身仍是数组级仿真。
完整记录在
[`results/imdb_general_vector_summary.json`](../results/imdb_general_vector_summary.json)
和
[`results/imdb_general_vector_qos_sweep.csv`](../results/imdb_general_vector_qos_sweep.csv)。

## 3. “优越性”现在具体体现在哪里

必须把三层证据分开：

| 层 | 目前得到的结论 | 是否已经实测 |
|---|---|---|
| 任务性能 | QOS 重建 learned weight 后，IMDb official-test accuracy 仍接近原 Ridge | 是；分类实测，QOS 为 active 数组仿真 |
| 空间记账 | 论文 QOS 数据访问/草图寄存器随维度按对数增长，而显式稀疏 classical model 需要保存大量非零项 | 公式记账；不是硬件测量 |
| 门级代价 | $D=4$ full QSVT 在本地 basis 编译后已有 251–444 个二比特门 | 是；本地编译资源，不是设备执行 |

[`assets/figures/imdb_accuracy_vs_machine_size_accounting.png`](../assets/figures/imdb_accuracy_vs_machine_size_accounting.png)
把同一组实测 accuracy 分别映射到 streaming feature dimension、训练稀疏矩阵
nonzeros 和论文 logical-qubit 公式。它用于说明“为什么空间复杂度值得研究”，
但不能把混合的理论横轴解释成真机资源优势。

两个具体记账点是：

| $D$ | measured accuracy | train nonzeros | 论文公式 logical qubits |
|---:|---:|---:|---:|
| 256 | 0.67744 | 3,093,016 | 49 |
| 16,384 | 0.85196 | 4,889,744 | 50 |

这里 49/50 是按论文公式和实测 sparsity 代入得到，不是把三百多万个 classical
numbers 宣称成可直接替换的 qubits。真正的端到端比较还必须加入 oracle 构造、
QSVT 深度、成功概率、误差修正和 classical control 的资源。

目前最强且严谨的表述是：

> 在真实 IMDb official split 上，general-vector QOS 的 active sketch 在有限样本下
> 保留了大部分分类性能；论文公式预测其空间资源按对数扩展。完整 $D=4$ QSVT
> 已编译成可提交线路，但系统级优势仍需真机和更大规模端到端实现验证。

## 4. 真机包

### 低风险入口：`09`

IMDb official-train 派生的 $D=4$ centroid direction 为

$$
(0.56927,-0.56689,-0.58299,-0.12120),
$$

对应 sign pattern `[+,-,-,-]`。notebook 没有为了得到该 pattern 调整 hash seed。

`09` 生成 65 条 2-qubit flat-QOS QASM，覆盖：

- H-only sanity check；
- exact-pair / balanced-stream control；
- IMDb IID stream 的 $M=4,16,32$，每个预算 20 条独立流；
- 与 IMDb 模型严格分开的 synthetic entangled control。

全部 QASM 已通过 round-trip parse、hash 和 direct-vs-QASM statevector 检查；
最大完整态误差为 $1.57\times10^{-15}$。
真机单元还会在提交前重新计算 hash、检查 2q/2c/2-measure schema，只接受在线
队列状态和正整数 task id；取回后自动保存 raw JSON、returned transpiled
QASM/hash，并计算 $F_{\rm echo}=P(00)$。

入口文件：

- [`results/imdb_vector_pilot/manifest.csv`](../results/imdb_vector_pilot/manifest.csv)
- [`results/imdb_vector_pilot/qasm/`](../results/imdb_vector_pilot/qasm/)

### 完整 general-vector 入口：`10`

`10` 使用 4 个逻辑 qubits：

- 2 address qubits；
- 1 QSVT signal qubit；
- 1 real-part LCU/herald qubit。

固定回归线路得到：

- $V$ unitarity spectral error：$4.43\times10^{-15}$；
- NumPy 与完整门级 postselected branch 的 $\ell_2$ 误差：
  $2.62\times10^{-15}$；
- success probability：0.0438886；
- conditional fidelity：0.996108。

Qiskit optimization level 3 的本地 basis-level 资源为：

| $M_{\rm fetched}$ | circuit | depth | two-qubit gates |
|---:|---|---:|---:|
| 8 | prepare | 408 | 251 |
| 8 | verify | 408 | 252 |
| 16 | prepare | 719 | 443 |
| 16 | verify | 719 | 444 |

这正是为什么必须先跑 `09` 的低深度 control，再决定是否把 `10` 的 deep
proof-of-principle 送上真机。

入口文件：

- [`results/general_vector_qsvt_pilot/manifest.json`](../results/general_vector_qsvt_pilot/manifest.json)
- [`results/general_vector_qsvt_pilot/qasm/`](../results/general_vector_qsvt_pilot/qasm/)
- [`results/general_vector_qsvt_pilot/gate_equivalence.json`](../results/general_vector_qsvt_pilot/gate_equivalence.json)

## 5. 推荐真机顺序

1. 在 `09` 中查询在线后端并运行一条 H-only sanity check；
2. 对同一条低深度 balanced/exact-pair control 比较可用编译后端；
3. 保存 raw counts、corrected counts、task id 和 returned transpiled QASM；
4. control 合格后运行一个 IMDb flat IID stream；
5. 只有成功事件数和编译资源仍可接受时，才运行 `10` 的
   `imdb_d4_general_qsvt_m8_verify`；
6. 报告 success events、heralding probability 和 conditional fidelity，
   不只报告条件 fidelity。

`09` 和 `10` 的提交开关都默认是 `False`，需要人工选定唯一 experiment id
后才能提交。`10` 还会在提交前重新计算 QASM hash，并强制检查 4 qubits、
4 classical bits 和 4 次 measurement。

## 6. 可复现运行顺序

```text
07_general_vector_qos
  → 08_imdb_general_vector_classification
  → 09_quafu_imdb_vector_pilot
  → 10_quafu_general_vector_qsvt_pilot
```

`08` 需要本地 aclImdb 数据。可设置 `IMDB_DATA_ROOT` 指向 `aclImdb` 目录；
数据说明见 [`data/README.md`](../data/README.md)。
