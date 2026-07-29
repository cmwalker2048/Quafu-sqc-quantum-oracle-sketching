# 原仓库审计与实现对应

核对仓库：

- 仓库：<https://github.com/haimengzhao/quantum-oracle-sketching>
- 审计日期：2026-07-29
- 固定快照：
  [`10c092cefcfdff9951bf5729bd2ffb4c25fe2254`](https://github.com/haimengzhao/quantum-oracle-sketching/tree/10c092cefcfdff9951bf5729bd2ffb4c25fe2254)
- 关键源文件：
  [`qos.py`](https://github.com/haimengzhao/quantum-oracle-sketching/blob/10c092cefcfdff9951bf5729bd2ffb4c25fe2254/qos.py)、
  [`qos_sampling.py`](https://github.com/haimengzhao/quantum-oracle-sketching/blob/10c092cefcfdff9951bf5729bd2ffb4c25fe2254/qos_sampling.py)、
  [`data_generation.py`](https://github.com/haimengzhao/quantum-oracle-sketching/blob/10c092cefcfdff9951bf5729bd2ffb4c25fe2254/data_generation.py)、
  [`qsvt.py`](https://github.com/haimengzhao/quantum-oracle-sketching/blob/10c092cefcfdff9951bf5729bd2ffb4c25fe2254/qsvt.py)、
  [`real_datasets/imdb_svm.py`](https://github.com/haimengzhao/quantum-oracle-sketching/blob/10c092cefcfdff9951bf5729bd2ffb4c25fe2254/real_datasets/imdb_svm.py)。

本表解决一个关键问题：原仓库中的 JAX 数组函数，与本项目实际生成的门级
Quafu-SQC 实验分别对应到哪里。

| 论文/功能 | 原仓库 | 本项目 | 状态 |
|---|---|---|---|
| 均匀采样 Boolean 数据 | `data_generation.boolean_data.get_data` | `01` 的 `draw_uniform_stream` | 已复现 |
| 显式采样 Boolean phase sketch | `qos_sampling.q_oracle_sketch_boolean` | `01` 的 `qos_phase_angles` | 已复现数学作用 |
| Boolean phase sketch 的门级实现 | 原仓库只返回 diagonal array | `03`/`05`/`06` 的 projector-phase QASM | 已新增 |
| `F_+` 验证 | 原仓库数值误差测试 | `01`/`02`/`03`/`05`/`06` 的 exact inverse echo | 已实现 |
| expected-unitary Boolean sketch | `qos.q_oracle_sketch_boolean` | `05` 的 `expected_diagonal` 与闭式 fidelity | 已实现并区分真实平均 |
| $n=3$ Boolean gate sketch | 原仓库返回 diagonal array | `05`/`06` 的 5-qubit、双工作 ancilla QASM | 已实现离线/压力测试 |
| Hardware campaign | 原仓库无 Quafu-SQC 任务层 | `06` 的多控制 QASM manifest 与结果解析 | 已生成，未提交 |
| flat $\pm1$ vector state sketch | `qos_sampling.q_state_sketch_flat_unitary` | `07` 数组/闭式回归；`09` IMDb-derived 2-qubit QASM | 已实现 |
| general-vector state sketch | `qos_sampling.q_state_sketch` | `07` active 数组；`08` learned IMDb weight；`10` full $D=4$ gate circuit | 已实现到小规模门级 |
| QSVT | `qsvt.apply_qsvt*` | `07` degree-4 数组；`10` signal + real 两辅助位线路 | 已实现 $D=4$，未做 amplitude amplification |
| IMDb ridge accuracy | `real_datasets/imdb_svm.py` 的 `RidgeClassifier` + CV | `08` 官方 25k/25k split、stateless hashing + Ridge | 已真实运行且修复原脚本泄漏 |
| 真机 orchestration | 原仓库无 Quafu-SQC 提交/取回层 | `11` bit-order controls、提交 guard、task registry、raw/returned-QASM parser | 已实现，默认 dry run |
| tiny Ridge inverse | 论文 Theorem F.13；仓库 `qsvt.py` 提供 QSVT utilities | `12` 的 2×2 normal-equation block encoding + degree-3 inverse QSVT | 已实现谱点精确 tiny 版本 |
| interferometric shadow | 论文 Lemma F.16；真实数据脚本未执行 | `12` 的干涉态、global-Clifford snapshots 与 held-out classification | 已实现 12-setting tiny pilot |
| Figure-2/tiny 对照 | 原脚本为同一经典 accuracy + 三种空间公式 | `12` 用 held-out hashing accuracy 重新记账，tiny 机制放独立面板 | 非逐点复现，避免混用资源轴 |

## Boolean 对应关系

原仓库显式采样代码的核心是：

```text
phase[x] += sampled_value
phase *= pi * dim / num_samples
diag = exp(i * phase)
```

本项目 `01` 保留同样的数学作用，但把变量命名拆得更清楚：

```text
theta = pi * N / M
phase[x_i] += theta * y_i
```

本项目 `03` 再把每一个 `phase[x_i] += theta*y_i` 展开成：

```text
X mask
→ cx/ccx equality flag
→ u1(theta) on work ancilla
→ uncompute
→ undo X mask
```

因此：

```text
qos_sampling.py 的 diagonal 数组
        ↓ 门级重新合成
03_quafu_boolean_qos_qasm.ipynb 的 OpenQASM
```

不是把 JAX 代码自动“编译”为 Quafu-SQC。

对应的单样本线路图：

![单样本 projector-phase 线路](../assets/circuits/projector-phase-update.png)

## 为什么不能直接宣称原仓库已经做了真机实验

### 1. `qos_sampling.py`

它显式生成随机样本，但返回的是 JAX array，例如 Boolean 模块返回
`diag = exp(i*phase)`。这适合数值验证，不是 `H/CX/CCX/U1` 门列表。

### 2. `qos.py`

它使用 expected-unitary 高效模拟。函数直接计算期望单步 unitary 的对角元，
再通过对数与指数拼接大量步骤。这是 benchmark 的数值捷径，不等同于在硬件上
按样本顺序执行随机门。

此外，随机 unitary 的平均一般不是 unitary。`05` 明确分开：

$$
F\!\left(\mathbb E[U]\right)
\quad\text{和}\quad
\mathbb E[F(U)].
$$

前者对应作者的 mean-operator surrogate；后者才对应大量独立随机 QASM
线路成功率的平均。两者都已用闭式公式和显式 Monte Carlo 核对。

### 3. `data_generation.py`

`boolean_data`、`vector_data` 和 `matrix_data` 对象内部保存完整 truth table、
vector 或 matrix，再从中抽样。所以它验证算法数学和样本 scaling，但单独运行
该类不能证明完整模拟程序只有 polylogarithmic classical memory。

### 4. `real_datasets/imdb_svm.py`

该脚本的 accuracy 来自 scikit-learn `RidgeClassifier` 的五折交叉验证。
量子曲线的机器大小来自公式：

$$
2\left\lceil\log_2(N_{\rm samples}+2D)\right\rceil
+\left\lceil\log_2(s+1)\right\rceil+4.
$$

它是“经典准确率 + 理论机器空间”的资源比较，不是 IMDb 数据经过完整
QOS、block encoding、QSVT 和 shadow readout 后得到的真机准确率。

本项目 `08` 不复用该有泄漏的 50,000 合并后 CV。它只在 official train
拟合，official test 只评估；同时真正把 learned Ridge weight 送入
general-vector QOS 数组路径。`09`/`10` 再从 official train 派生 $D=4$
centroid vector，分别生成 flat 与 full general-vector QASM。

`12` 进一步用 official-train-only 的监督词汇投影构造两个特征，再真正运行
2×2 Ridge inverse QSVT 和干涉式 shadows。该投影的经典存储没有计入 4-qubit
线路，因此 tiny 点只用于机制验证，不能加入原 Figure 2 的同一 machine-size
纵轴。简单 polarity rule 与 exact tiny Ridge 的 accuracy 几乎相同；QSVT
证据来自 heralding/verifier，而不是 accuracy lift。

## General-vector 数组与门级对应

`07` 发现并公开固定源码的三个重要实现细节：

1. active 路径同时在 sampled values 和 scale 中除以 norm；公开 wrapper 先把
   目标变成单位向量，恢复尺度不变性；
2. degree-4 生成 4 组样本，但只有 3 次 signal call，因此 25% fetched 样本
   没有进入 QSVT；
3. expected 使用 $1/5$，active 使用 $1/3$，二者不是同一有限样本 estimator。

`10` 把 active 的每个样本编译为

$$
\exp\left(i\alpha_\ell Y_{\rm signal}Z^{j_\ell}\right),
$$

用 degree-4 QSVT 得到 $V$，再用第二个辅助位实现

$$
\frac{V+V^\dagger}{2}.
$$

固定 IMDb-derived $D=4$ 测试中，NumPy postselected branch 与完整 4-qubit
statevector 的 $\ell_2$ 误差约为 $2.62\times10^{-15}$。这完成了小规模
gate mechanism 对应，但静态 QASM 仍不是在线 stream controller。

## 对后续实现的约束

1. 每个新 notebook 必须写出它对应的原仓库函数；
2. 数值矩阵模拟与门级线路必须分栏报告；
3. 使用 expected unitary 时必须显式标记；
4. 使用理想 state preparation 时必须显式标记；
5. 真机结果必须保存编译后线路，不能只报告逻辑级 MCX；
6. 不把静态 QASM 上传称为严格的在线 streaming controller；
7. generic Aer noise 不能称为 Quafu-SQC calibrated noise；
8. $n=3$ 的 qubit 数仍小，但每个有效 projector 已需 4 个 CCX；必须同时
   报告门数与编译后资源。

## 直接链接

- [仓库 README](https://github.com/haimengzhao/quantum-oracle-sketching/blob/main/README.md)
- [`qos_sampling.py`](https://github.com/haimengzhao/quantum-oracle-sketching/blob/main/qos_sampling.py)
- [`qos.py`](https://github.com/haimengzhao/quantum-oracle-sketching/blob/main/qos.py)
- [`data_generation.py`](https://github.com/haimengzhao/quantum-oracle-sketching/blob/main/data_generation.py)
- [`qsvt.py`](https://github.com/haimengzhao/quantum-oracle-sketching/blob/main/qsvt.py)
- [`real_datasets/imdb_svm.py`](https://github.com/haimengzhao/quantum-oracle-sketching/blob/main/real_datasets/imdb_svm.py)
