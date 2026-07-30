# 研究计划与验收标准

## 当前路线决定

完整复现原 Figure 2 不再是本阶段前置条件。其关键代码/资源含义已在
`SOURCE_MAP.md` 审计，完整 IMDb/PBMC、classification/PCA 复跑移到未来发布版
附录。

当前主线已经推进为：

$$
\text{Boolean 解析平均}
\rightarrow
\text{QASM 等价}
\rightarrow
\text{general-vector QOS}
\rightarrow
\text{官方 IMDb classification}
\rightarrow
\text{flat/full-vector Quafu-SQC pilot}
\rightarrow
\text{one-hot-calibrated hardware validation}
\rightarrow
\text{tiny Ridge inverse-QSVT + shadows}.
$$

去重依据与本轮结果见 `NEXT_STAGE_DECISION.md`。

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
- 当前 Quafu-SQC 物理机的门、退相干和读出噪声。

有限规模真机实验只能支持第三类中的局部、经验性结论，不能证明任意物理噪声
和任意规模下都保持指数优势。

## 阶段 0：环境、安全和可复现性

产出：

- 环境版本检查；
- 不含凭据的 Quafu-SQC 连接模板；
- 固定随机种子；
- notebook 执行顺序；
- Git 忽略规则。

验收：

- 十三个 notebook 从 `QuantumComputing` 环境可依次执行；
- 仓库文本与 notebook 源码中不存在 API token；
- 默认执行不会提交云端任务。

## 阶段 1：Boolean QOS primitive

目标：

$$
O_f=\sum_x(-1)^{f(x)}|x\rangle\langle x|.
$$

本轮已执行参数：

- 门级地址量子比特 $n=1,2,3$；
- $N=2^n$；
- parity、majority、稀疏 marked set、随机 Boolean function；
- $M=N,2N,4N,\ldots$；
- 解析平均检验使用独立数据流 $R_{\rm streams}=1200$。

$n=4$ 保留为后续 NumPy/资源扩展点，不冒充本轮已有的可审计门级证据。

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
- 作者 expected operator 与真实 $\mathbb E[F_{\rm echo}]$ 被分开；
- expected operator、解析 fidelity 平均与显式 Monte Carlo 相互核对；
- $n=1,2,3$ 的 QASM 完整 statevector 与直接相位公式误差小于 $10^{-10}$，
  并包含非对称 random truth table 的端序检查。

## 阶段 2：噪声与有限最优样本数

先实现两个层次：

1. `02` 的可解释教学模型：
   - coherent phase bias；
   - per-update phase jitter；
   - local phase-flip/dephasing；
   - symmetric readout bit flip；
   - finite shots。
2. `06` 的实际门级 generic 模型：
   - 先把 CCX 分解到 `u` + `cx`；
   - 每个一、二比特门施加 depolarizing error；
   - measurement 施加 symmetric readout error；
   - balanced-stream control 隔离 IID 计数波动与门噪声；它匹配同一 $M$ 和
     平均门数尺度，不声称与每条 IID 线路严格等深。

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
- 清楚标出扫描的 $R_{\rm streams},M_{\rm data},S_{\rm shots}$；
- 不把简化密度矩阵模型称为 Quafu-SQC 的完整设备噪声模型；
- 如果曲线出现最低点，先对独立 stream 做 bootstrap；
- 只有内部最优点达到预先规定的 80% bootstrap **选择频率启发式**时，才报告
  稳定候选 $M_\mathrm{opt}$；该频率不解释成“真实最优概率”，否则报告完整曲线。
- generic Aer 模型不能称为 Quafu-SQC calibrated noise。

## 阶段 3：Quafu-SQC 门级功能验证

主真机 pilot 优先 $n\le2$。离线 campaign 已扩展到 $n=3$：

- 使用两个 clean work qubits，总共 5 qubits；
- 每个有效 projector update 使用 4 个 CCX；
- 真机只先考虑 sparse marked case；
- balanced/majority 只作为编译与门数压力测试。

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
- QASM 的逻辑门计数；
- H-only、exact-pair、balanced-stream 和 IID 四类主线路，另加
  compiler-stress 与 optimized twin；
- barrier-preserving 主线路与少量 no-barrier optimized twins；
- optimized twin 必须复用同一条 balanced stream，并在 manifest 记录 pair id；
- manifest 中的 truth table、完整 stream、hash、shots 与门预算。

验收：

- QASM 可在 notebook 中完整打印和保存；
- 提交前输出门计数、qubit 数、`M` 和 QASM 哈希；
- `SUBMIT_TO_HARDWARE=False` 为默认值；
- 必须显式启用小批 experiment plan，默认每次最多两条，不能默认提交 campaign；
- 后端状态必须实时查询，不能使用旧 demo 的截图；
- 任务结果保存 `tid`、原始 counts、transpiled QASM 和实验参数。

`11` 进一步加入：

- 2q/4q one-hot bit-order controls；
- 只接受非负 queue 状态和正整数 task id；
- duplicate submission 与 deep-QSVT guards；
- task registry、非终态 result 检查、raw/returned QASM 持久化；
- flat/general 指标的 95% Wilson 区间。

局限：

- 静态 QASM 预先保存所有样本门，因此只验证 gate-level dynamics；
- `F_+` 只验证 $|+\rangle^{\otimes n}$ 输入；
- 编译后的 `ccx` 会增加很多二比特门；
- barrier 保留静态样本边界，但不会把 QASM 变成在线 streaming。

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

`04` 实现前四项，用来隔离“读出是否正确”；它仍不是完整 QOS 分类器。
后续证据链已经补齐：

- `07`：general-vector expected/active QOS 与 fixed-source regression；
- `08`：官方 IMDb 25k/25k、真实 Ridge weight、QOS 重建后重新分类；
- `09`：IMDb-derived flat sign-vector QASM；
- `10`：IMDb-derived $D=4$ full degree-4 QSVT + real-part LCU QASM。

`08` 的训练仍是经典 Ridge，`09`/`10` 的低维 target 是 class-centroid。
因此当前结果应称为“真实数据上的 vector-sketch classification bridge”，
而不是完整量子 LS-SVM 训练。

## 阶段 5：tiny quantum ridge solver

目标：

$$
w=(X^TX+\lambda I)^{-1}X^Ty,
\qquad
g(\sigma)=\frac{\sigma}{\sigma^2+\lambda}.
$$

`12` 已完成第一版：

- official IMDb train 学得两个监督式聚合特征，$D=2$；
- average-loss normal equation $A=X^\top X/N+\lambda I$；
- 2×2 Halmos block encoding；
- 只在两个已知谱点精确的 degree-3 odd inverse polynomial；
- real-part LCU 与 postselection；
- 4-qubit dense-resynthesized hardware circuit。

已通过的验收：

- vector state sketch 已被多输入态测试，并已由 `10` 做 $D=4$ 门级等价；
- block encoding 的投影块与目标矩阵数值一致；
- 多项式谱点误差与线路误差分开报告；
- ideal heralding probability 0.10457；
- postselected weight-state fidelity 数值上约为 1；
- prepare/verify 本地编译各 18 CX。

这里的 QSVT 指 ridge inverse polynomial；`07`/`10` 已实现的是 vector state
sketch 内部用于 arcsin 的 QSVT，不能把二者混为同一个完成项。

边界：degree-3 polynomial 不是连续谱区间上的通用 inverse approximation；
dense unitary resynthesis 的经典成本指数增长；当前使用 postselection，没有
amplitude amplification。正规方程条件数约 2.204，对应论文
singular-value 口径的 $\kappa_{\rm reg}\approx1.485$。

## 阶段 6：classical shadows

`12` 已完成 tiny pilot：

- controlled state preparation；
- two-qubit global-Clifford ensemble；
- 保存随机 seed 和 bitstring；
- 12 个 distinct settings、每个 2,048 shots；
- 干涉态保存 $\operatorname{Re}\langle x|w\rangle$ 的符号；
- 对全部 25,000 个 test 点重新分类；
- 分阶段生成 shallow-control 与 full-QSVT-shadow 真机包。

当前 shadow 会显式重建 $4\times4$ density estimator，只适合 $D=2$ 自检；
distinct-setting cluster bootstrap 是探索性区间，不是论文
i.i.d.-Clifford + median-of-means 保证。12-setting Aer QSVT-shadow 的分类
accuracy 约 0.849，但 weight norm ratio 约 0.623、signed-overlap RMSE
约 0.303，因此应称为 sign/classification pilot，而非
$\epsilon$-accurate overlap demonstration。

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

当前已完成第 1 项的解析/Monte Carlo/generic-noise 版本；第 2 项已有官方 IMDb
实测准确率与 formula-only logical-space 曲线；第 4 项已有本地编译前后资源。
真实设备相关部分必须等 Quafu-SQC result 与远端 transpiled circuit 返回后再填入。
