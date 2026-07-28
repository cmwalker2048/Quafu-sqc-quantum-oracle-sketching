# 原仓库审计与实现对应

核对仓库：

- https://github.com/haimengzhao/quantum-oracle-sketching

本表解决一个关键问题：原仓库中的 JAX 数组函数，与本项目实际生成的门级
Quafu 实验分别对应到哪里。

| 论文/功能 | 原仓库 | 本项目 | 状态 |
|---|---|---|---|
| 均匀采样 Boolean 数据 | `data_generation.boolean_data.get_data` | `01` 的 `draw_uniform_stream` | 已复现 |
| 显式采样 Boolean phase sketch | `qos_sampling.q_oracle_sketch_boolean` | `01` 的 `qos_phase_angles` | 已复现数学作用 |
| Boolean phase sketch 的门级实现 | 原仓库只返回 diagonal array | `03` 的 `append_projector_phase_qasm` | 已新增 |
| `F_+` 验证 | 原仓库数值误差测试 | `01`/`02` 与 `03` 的 exact inverse 验证 | 已实现 |
| expected-unitary Boolean sketch | `qos.q_oracle_sketch_boolean` | 暂无单独 notebook | 待作为理论对照加入 |
| flat $\pm1$ vector state sketch | `qos_sampling.q_state_sketch_flat_unitary` | 暂无 | 下一阶段 |
| general-vector state sketch | `qos_sampling.q_state_sketch` | `04` 只定义读出接口，未替换理想态制备 | 待实现 |
| QSVT | `qsvt.apply_qsvt*` | 暂无门级 Quafu 合成 | 待实现 |
| IMDb ridge accuracy | `real_datasets/imdb_svm.py` 的 `RidgeClassifier` + CV | 暂不复现 | 经典基线/资源公式，不是真机线路 |

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

不是把 JAX 代码自动“编译”为 Quafu。

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

## 对后续实现的约束

1. 每个新 notebook 必须写出它对应的原仓库函数；
2. 数值矩阵模拟与门级线路必须分栏报告；
3. 使用 expected unitary 时必须显式标记；
4. 使用理想 state preparation 时必须显式标记；
5. 真机结果必须保存编译后线路，不能只报告逻辑级 MCX；
6. 不把静态 QASM 上传称为严格的在线 streaming controller。

## 直接链接

- [仓库 README](https://github.com/haimengzhao/quantum-oracle-sketching/blob/main/README.md)
- [`qos_sampling.py`](https://github.com/haimengzhao/quantum-oracle-sketching/blob/main/qos_sampling.py)
- [`qos.py`](https://github.com/haimengzhao/quantum-oracle-sketching/blob/main/qos.py)
- [`data_generation.py`](https://github.com/haimengzhao/quantum-oracle-sketching/blob/main/data_generation.py)
- [`qsvt.py`](https://github.com/haimengzhao/quantum-oracle-sketching/blob/main/qsvt.py)
- [`real_datasets/imdb_svm.py`](https://github.com/haimengzhao/quantum-oracle-sketching/blob/main/real_datasets/imdb_svm.py)
