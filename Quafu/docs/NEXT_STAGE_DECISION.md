# 下一阶段取舍：先闭合 Boolean QOS，再进入真实数据

> 历史决策记录：本文件描述的是 `05`/`06` 开始前的取舍。该阶段已经完成；
> general-vector、官方 IMDb、flat QASM 与 full $D=4$ QSVT 已分别在
> `07`–`10` 实现。当前结果总览以 README 和
> `GENERAL_VECTOR_IMDB_RESULTS.md` 为准。

## 结论

本轮不完整复现论文 Figure 2，也不立刻做 tiny LS-SVM/QSVT 或
classical shadows。当前最有研究价值的目标是：

> 把 Boolean QOS primitive 从“数学上看起来合理”推进到
> “解析平均、显式随机流、实际 QASM、门级噪声和 Quafu-SQC 实验包彼此可核对”。

因此本轮新增：

- `05_boolean_qos_equivalence.ipynb`；
- `06_quafu_experiment_bundle.ipynb`。

## 一分钟阅读顺序

如果只想知道“现在做到哪、下一步干什么”，按这个顺序：

1. 看本文件的“去重后的决策表”；
2. 打开 `05`，先读开头符号表，再看 expected operator 与完整 statevector 检验；
3. 打开 `06`，先读五类控制实验表，再看 manifest 和两张工程图；
4. 真机路线按 H-only → 同线路双编译器 A/B → 保存原始/transpiled 结果推进；
5. 无需 token 的 flat $\pm1$ vector sketch 可以与真机路线并行离线推进。

## 为什么 Figure 2 不在关键路径

原 Figure 2 的任务性能来自经典 `RidgeClassifier` 或 SVD，量子部分是
logical-qubit 空间公式。完整复跑的价值是核对数据预处理、资源公式和作图，
但它不会回答当前最大的未知量：

- sampled QOS 与作者 expected-unitary surrogate 是否一致；
- 手写 QASM 是否真的实现了目标相位；
- $M$ 增大时采样收益能否抵消门数和物理噪声；
- 编译后线路是否仍在当前硬件可运行范围内。

现有 `SOURCE_MAP.md` 已经完成 Figure 2 的关键解释性审计。完整 IMDb/PBMC、
classification/PCA 四面板复跑保留为未来发布版附录，不作为真机实验前置条件。

## 去重后的决策表

| Pro 计划项 | 处理 | 理由 |
|---|---|---|
| 统一 Quafu-SQC / `quafusqc` 术语 | 已完成，保留 | 当前本机接口已验证 |
| 完整复现 Figure 2 | 延后到附录 | 主要是经典计算与理论空间公式 |
| Boolean 显式采样 | 已由 `01` 完成 | 不重复写基础教程 |
| 简化噪声教学模型 | 已由 `02` 完成 | 保留，但不冒充设备模型 |
| 单条 $n=2$ QASM | 已由 `03` 完成 | 作为 smoke test 基础 |
| expected-unitary 对照 | 本轮完成 | 是作者代码与门级实验之间的关键缺口 |
| QASM/完整 statevector 等价 | 本轮完成 | 排除端序、相位和反计算错误 |
| $n=3$ | 只做离线与 sparse 压力测试 | 每个有效 update 已需 4 个 CCX |
| Quafu-SQC campaign | 本轮生成，默认不提交 | token 已暴露过，且需先做低深度 pilot |
| 真实 IMDb | 延后 | 先实现 vector-QOS，避免只得到经典 centroid + 理想态 |
| tiny QSVT / shadows | 延后 | 依赖 vector sketch 与稳定读出 |

## 本轮已完成的证据

### 1. expected operator 与真实平均被严格分开

作者 `qos.py` 使用的是平均算子。它通常不是 unitary，而且通常有

$$
F\!\left(\mathbb E[U]\right)
\neq
\mathbb E[F(U)].
$$

`05` 同时实现：

- expected operator 的闭式表达；
- 真正 $\mathbb E[F_{\rm echo}]$ 的闭式表达；
- 1,200 条显式 IID stream 的 Monte Carlo 检验。

最大 Monte Carlo fidelity 偏差为约 $2.69$ 个标准误，处于统计波动范围。

### 2. 实际 QASM 与直接相位公式等价

`05` 对 $n=1,2,3$、parity/marked/majority/random、balanced/IID stream
逐条执行 Qiskit statevector。它不只比较 $F_{\rm echo}$：还从 residual phase
直接构造完整解析输出态，对齐无物理意义的全局相位后逐振幅比较。结果：

$$
\max \left\|
|\psi_{\rm direct}\rangle-
e^{i\phi}|\psi_{\rm QASM}\rangle
\right\|_2
\approx 9.16\times 10^{-16},
$$

最大 fidelity 标量差约 $1.33\times10^{-15}$，工作 ancilla 泄漏为 $0$，
balanced stream 的 $F_{\rm echo}$ 在数值精度内为 1。这些 $10^{-15}$ 量级残差
是浮点舍入，不是真机误差。

### 3. hardware-aware campaign 已可审计

`06` 生成 42 条可解析 QASM，包含：

- H-only；
- exact oracle → inverse；
- balanced-stream control（内部历史字段仍为 `balanced_depth`）；
- IID streams；
- no-barrier optimized twins；
- $n=3$ sparse/stress cases。

每条任务保存 truth table、完整 stream、三类 hash、逻辑门数、CX proxy、
shots 和安全任务模板。

### 4. 没有过度声称 $M_{\rm opt}$

generic depolarizing + readout 模型展示了采样收益与门噪声的竞争。对独立
stream 重抽样后，$M/N=8$ 成为样本均值最优点的频率约为 64.4%；这不是“它是
真实最优点的概率”。由于没有达到预先规定的 80% 启发式筛选阈值，因此当前结论是：

> 这套通用模型出现交叉趋势，但尚不能报告稳健的内部 $M_{\rm opt}$。

它不是 Quafu-SQC 校准噪声，也不是硬件实测结果。

## 下一主线

真机路线按以下固定顺序推进：

1. 确认凭据只通过环境变量或隐藏输入提供；
2. 先跑 H-only sanity check；
3. 对同一条低深度 exact-pair 或 balanced-stream 线路，用 `quarkcircuit` 与
   `qsteed` 做 A/B；
4. 保存完整 raw JSON 和 transpiled QASM，比较全零概率、二比特门数、二比特
   depth 以及是否仍有三比特门；
5. 冻结编译器，再决定是否扩大 IID/$M$。

这条路线不阻塞离线主线。无需 token 即可并行推进：

$$
\text{flat }\pm1\text{ vector sketch}
\longrightarrow
\text{general-vector QOS}
\longrightarrow
\text{替换 `04` 的理想 }|w\rangle
\longrightarrow
\text{small IMDb}
\longrightarrow
\text{tiny ridge/QSVT}.
$$

真实 IMDb 阶段应使用官方 train/test split、无词表的 signed feature hashing
作为主表示，并且不把经典 `StatePreparation` 误称为 QOS state preparation。
