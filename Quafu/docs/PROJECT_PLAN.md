# 研究计划与验收标准

## 研究问题

本项目不把“量子加速”笼统地当作一个指标，而是分别回答：

1. QOS 的 Boolean oracle primitive 能否被重新写成可执行门级线路？
2. 随着经典样本数 $M$ 增加，oracle 误差是否按预期下降？
3. 真实硬件噪声是否会造成有限最优样本数，而不是 $M$ 越大越好？
4. QOS sketch 能否服务于一个端到端的小型学习任务？
5. 理论 logical-space separation 与当前硬件工程资源之间差多少？

## 研究边界

必须区分三种“噪声下仍有优势”的说法：

- 论文讨论的 noisy/correlated **classical data**；
- 容错模型中的 logical quantum noise；
- 当前 Quafu 物理机的门、退相干和读出噪声。

有限规模真机实验只能支持第三类中的局部、经验性结论，不能证明任意物理噪声
和任意规模下都保持指数优势。

## 阶段 0：环境、安全和可复现性

产出：

- 环境版本检查；
- 不含凭据的 Quafu 连接模板；
- 固定随机种子；
- notebook 执行顺序；
- Git 忽略规则。

验收：

- 五个 notebook 从 `QuantumComputing` 环境可依次执行；
- 仓库文本与 notebook 源码中不存在 API token；
- 默认执行不会提交云端任务。

## 阶段 1：Boolean QOS primitive

目标：

$$
O_f=\sum_x(-1)^{f(x)}|x\rangle\langle x|.
$$

实验参数：

- 地址量子比特 $n=2,3,4$；
- $N=2^n$；
- parity、majority、稀疏 marked set、随机 Boolean function；
- $M=N,2N,4N,\ldots$；
- 独立数据流 $K\ge 100$。

核心指标：

$$
F_+=\left|
\frac1N\sum_x
\exp\left(i[\widetilde\phi_x-\pi f(x)]\right)
\right|^2.
$$

验收：

- 代码用计数直接验证单个地址的相位累计；
- `1 - F_+` 随 $M$ 总体下降；
- 图中显示 between-stream 置信区间；
- finite shots 被作为第二层随机性单独加入。

## 阶段 2：噪声与有限最优样本数

先实现可解释的简化模型：

- coherent phase bias；
- per-update phase jitter；
- local phase-flip/dephasing；
- symmetric readout bit flip；
- finite shots。

研究假设：

$$
\epsilon_{\mathrm{tot}}(M)
\approx
\frac{aN}{M}
+bMg(n)p_{\mathrm{gate}}
+\epsilon_{\mathrm{readout}}.
$$

验收：

- 同图比较 ideal、coherent error、dephasing/readout；
- 清楚标出扫描的 $K,M,S$；
- 不把简化密度矩阵模型称为 Quafu 的完整设备噪声模型；
- 如果曲线出现最低点，报告经验 $M_\mathrm{opt}$ 及不确定性。

## 阶段 3：Quafu 门级功能验证

第一版限制为 $n\le 2$，原因是 OpenQASM 2.0 的 `ccx` 可以直接表达，
而更高阶 MCX 需要额外 ancilla 与明确的分解策略。

单样本门：

1. X mask 把 $|x_i\rangle$ 映射到全 1；
2. `cx`/`ccx` 计算 equality flag；
3. ancilla 上 `u1(theta)`；
4. 反计算 equality flag；
5. 撤销 X mask。

线路：

```text
H^n → sampled projector phases → exact inverse → H^n → measure
```

每个核心门级 notebook 同时提供：

- 可读的逻辑概览图；
- 从实际 QASM 解析出的逐门线路图；
- 保存到 `assets/circuits/` 的 PNG；
- QASM 的逻辑门计数。

验收：

- QASM 可在 notebook 中完整打印和保存；
- 提交前输出门计数、qubit 数、`M` 和 QASM 哈希；
- `SUBMIT_TO_HARDWARE=False` 为默认值；
- 后端状态必须实时查询，不能使用旧 demo 的截图；
- 任务结果保存 `tid`、原始 counts、transpiled QASM 和实验参数。

局限：

- 静态 QASM 预先保存所有样本门，因此只验证 gate-level dynamics；
- `F_+` 只验证 $|+\rangle^{\otimes n}$ 输入；
- 编译后的 `ccx` 会增加很多二比特门。

## 阶段 4：小型 centroid 分类

模型：

$$
w=\mathbb E[yx], \qquad
\widehat y(x')=\operatorname{sign}\operatorname{Re}\langle x'|w\rangle.
$$

工程顺序：

1. 经典 streaming accumulator 作为 oracle；
2. 理想 $|w\rangle$ 制备；
3. Hadamard-test 的 finite-shot 读出；
4. 报告 accuracy 和 margin；
5. 再用真正的 vector QOS state sketch 替换理想态制备。

当前 `04` notebook 实现前四项，用来隔离“读出是否正确”。它不是完整 QOS
分类器，必须保留这一说明。

## 阶段 5：tiny ridge 与 QSVT

目标：

$$
w=(X^TX+\lambda I)^{-1}X^Ty,
\qquad
g(\sigma)=\frac{\sigma}{\sigma^2+\lambda}.
$$

第一版：

- $D=4$；
- 训练样本 4–8；
- 每行 1–2 个非零元素；
- QSVT degree 3 或 5；
- 模拟器优先；
- 真机只提交编译后仍可控的子电路。

验收前置条件：

- vector state sketch 已被多输入态测试；
- block encoding 的投影块与目标矩阵数值一致；
- 多项式逼近误差与线路误差分开报告。

## 阶段 6：classical shadows

只有 Hadamard-test 分类已经稳定后才开始：

- controlled state preparation；
- 随机 Clifford/合适的 shadow ensemble；
- 保存随机 seed 和 bitstring；
- 多 observable 置信区间；
- 多测试点摊销成本。

## 经典基线和公平比较

基线分为：

- full sparse ridge；
- online SGD / passive-aggressive / online ridge；
- feature hashing / count sketch / sparse JL；
- 任务适用时使用 frequent directions。

每个实验必须写明：

- 数据是否 IID；
- 是否允许打乱和重复读取；
- 双方经过几遍数据；
- 是否允许保存历史样本；
- 数值精度；
- 峰值内存的测量口径。

分别制作：

1. 固定目标精度，比较最小机器空间；
2. 固定机器空间，比较预测性能。

## 最终图表

至少包含：

1. `oracle error vs number of data samples`；
2. `accuracy vs machine size`；
3. `accuracy vs physical noise strength`；
4. 编译前后的一、二比特门数和 depth；
5. $K,M,S$ 的层次化不确定性。
