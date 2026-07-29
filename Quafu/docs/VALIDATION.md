# 本地验证记录

验证日期：2026-07-29  
验证环境：`/opt/miniconda3/envs/QuantumComputing/bin/python`  
Python：3.13.7  
quafusqc：3.3.9  
Qiskit：2.5.1  
Qiskit Aer：0.17.2  
scikit-learn：1.9.0
JAX/JAXLIB：0.8.1
pyqsp：0.2.0

## 执行方法

十一个 notebook 均通过该环境的 Jupyter kernel 执行并保留输出。
所有联网和真机提交开关保持 `False`。

Markdown 与 notebook Markdown 单元的数学定界符统一为：

- 行内公式：`$...$`
- 独立公式：`$$...$$`

## 结果

| Notebook | 结果 |
|---|---|
| `00_environment_and_quafu_access` | 环境/import/QASM dry run 通过；无网络请求 |
| `01_boolean_qos_core` | 通过；`M=8→1024` 时平均 `1-F_+` 从约 0.740 降到 0.032 |
| `02_sampling_and_noise_benchmark` | 通过；教学 phase-flip 网格 argmin 为 256，未作稳健最优声明 |
| `03_quafu_boolean_qos_qasm` | 通过；生成 3-qubit QASM，平衡 smoke test 的理想 `F_+=1` |
| `04_centroid_readout_bridge` | 通过；理想重叠分类 accuracy 约 0.877 |
| `05_boolean_qos_equivalence` | 通过；expected operator/真实平均/Monte Carlo 与 $n=1,2,3$ 完整 statevector 等价 |
| `06_quafu_experiment_bundle` | 通过；42 条 QASM、generic Aer noise、manifest 与 mock result parser |
| `07_general_vector_qos` | 通过；fixed-source NumPy 回归、active/expected、heralding、bootstrap 与源码陷阱审计 |
| `08_imdb_general_vector_classification` | 通过；official 25k/25k Ridge 与 learned-weight QOS classification |
| `09_quafu_imdb_vector_pilot` | 通过；IMDb-derived flat target、65 条 2-qubit QASM |
| `10_quafu_general_vector_qsvt_pilot` | 通过；4-qubit full QSVT + real LCU、逐振幅等价与 4 条 QASM |

所有 notebook 的 error output 数量均为 0。

## 线路图产物

Qiskit 从实际 QASM 或对应的逻辑线路对象生成：

- `assets/circuits/bell-circuit.png`
- `assets/circuits/projector-phase-update.png`
- `assets/circuits/qos-verification-overview.png`
- `assets/circuits/qos-full-n2.png`
- `assets/circuits/hadamard-overlap-test.png`
- `assets/circuits/qos-equivalence-n1.png`
- `assets/circuits/projector-phase-update-n3.png`
- `assets/circuits/qos-campaign-balanced-n1.png`
- `assets/circuits/general-vector-pauli-sample.png`
- `assets/circuits/imdb-flat-qos-echo-overview.png`
- `assets/circuits/imdb-d4-general-qsvt-logical.png`

数值研究图：

- `assets/figures/expected-unitary-validation.png`
- `assets/figures/qasm-equivalence-validation.png`
- `assets/figures/qos-gate-pressure.png`
- `assets/figures/hardware-aware-qos-tradeoff.png`
- `assets/figures/general-vector-qos-scaling.png`
- `assets/figures/imdb_hashing_accuracy_and_formula.png`
- `assets/figures/imdb_general_vector_qos_sweep.png`
- `assets/figures/imdb-d4-flat-qos-results.png`
- `assets/figures/imdb-d4-general-qsvt-scaling.png`

其中 `qos-full-n2.png` 由 `qiskit.qasm2.loads(qasm_text)` 解析后绘制，因此也构成
一次独立的本地 QASM 语法检查。

## QASM 产物

`03` 生成：

- `results/boolean_qos_n2.qasm`
- `results/boolean_qos_n2_metadata.json`

逻辑级资源：

- 2 个地址 qubits；
- 1 个工作 ancilla；
- 36 个 `ccx`；
- 18 个 `u1`；
- 36 个 `x`；
- 4 个 `h`；
- 2 个 measurement。

这是编译前的逻辑门数。远端编译后的二比特门、SWAP 和 depth 只能在真实任务
返回的 `transpiled` 字段中统计。

## 新阶段数值验收

`05`：

- 闭式结果固定回归点：5 个全部通过；
- 显式 Monte Carlo 与解析 $\mathbb E[F_{\rm echo}]$ 的最大偏差：
  约 2.69 个标准误；
- Monte Carlo 平均算子与 expected operator 的最大归一化偏差：
  约 2.84 个估计标准误；
- 最大 `direct phase formula` vs QASM 的 $F_{\rm echo}$ 标量差：
  $1.33\times10^{-15}$；
- 全局相位对齐后的完整 statevector 最大 $\ell_2$ 误差：
  $9.16\times10^{-16}$；
- 最大单振幅误差：$8.88\times10^{-16}$；
- 最大 work-ancilla leakage：0；
- balanced-stream 最大偏离 1：$1.33\times10^{-15}$。

`06`：

- 生成并解析 42 条 campaign QASM；
- 六个 control type 包括 H-only、exact-pair、balanced-stream、IID、
  compiler-stress 和 optimized twin；
- 两条 optimized twin 已验证复用配对 balanced stream，区别只在 update barriers；
- generic-noise 扫描使用 $R_{\rm streams}=30$ 条独立流、每条
  $S_{\rm shots}=2048$；
- bootstrap 重抽样中 $M/N=8$ 的选择频率为 64.4%，低于预先规定的 80%
  启发式阈值，因此**没有报告稳健
  $M_{\rm opt}$**；
- 未被选中的 $M/N=16$ 也以 0 频率保存在结果表中；
- 所有硬件提交和真实结果获取开关保持关闭；
- mock parser 通过；二量子位 barrier 不再被误计为二比特门，但 mock 数据不是
  硬件结果。

`07`：

- NumPy expected vs fixed JAX source 最大误差：$2.05\times10^{-15}$；
- NumPy active vs fixed JAX source 最大误差：$9.19\times10^{-17}$；
- $D=64$ active mean conditional fidelity：
  $0.465$（$M_{\rm fetched}=256$）到
  $0.979$（$M_{\rm fetched}=16384$）；
- 对应 mean heralding probability：$0.0731$ 到 $0.0447$；
- signal-block 最大 unitarity 误差：$2.50\times10^{-16}$；
- normalized wrapper 的尺度回归误差：$6.66\times10^{-17}$；
- degree-4 固定源码 unused fetched fraction：25%；
- 所有 active summary 含 2,000 次 stream bootstrap 区间。

`08`：

- 官方 train/test：25,000 / 25,000，类别各半；
- signed HashingVectorizer + Ridge 的 official-test accuracy：
  $D=256$ 为 0.67744，$D=16384$ 为 0.85196；
- general-vector QOS 主 sweep 使用单位 learned-weight direction，再乘回
  $\|w\|_2$ 并保留 intercept 做分类；
- 最大 active 预算为 16,384 fetched / 12,288 QSVT-consumed samples；
- 20 条独立 active streams 的 success probability 为
  $0.04769\pm0.00190$，条件 fidelity 为
  $0.92796\pm0.01098$；
- 重新预测全部 official test 后的 accuracy 为
  $0.67165\pm0.00213$，原 $D=256$ Ridge 为 0.67744；
- accuracy 是真实 held-out prediction；logical-qubit 曲线仅为原论文公式。

`09`：

- official-train 派生 $D=4$ centroid sign pattern 为 `[1,-1,-1,-1]`，未调整
  hash seed 来制造该 pattern；
- 65 条 QASM 全部 parse/hash 通过；
- 最大 direct-vs-QASM 完整态误差：$1.57\times10^{-15}$；
- 另有严格分离的 synthetic entangled calibration control。
- 提交前强制复核 QASM hash 与 2q/2c/2-measure schema，只接受在线 queue
  状态和正整数 task id；
- 真实结果闭环保存 receipt、raw JSON、returned transpiled QASM/hash，并解析
  $F_{\rm echo}=P(00)$；提交与取回默认关闭。

`10`：

- 4 logical qubits：2 address + signal + real；
- 完整 NumPy-vs-gate postselected branch $\ell_2$ 误差：
  $2.62\times10^{-15}$；
- $V$ unitarity spectral error：$4.43\times10^{-15}$；
- Qiskit optimization level 3 后，$M=8$ prepare/verify 分别约
  251/252 个二比特门，$M=16$ 为 443/444；
- 4 条 QASM 均完成 round-trip parse 与 ideal 8,192-shot Aer 检查；
- 每条 manifest 记录并复核 target/QASM hash、固定源码 commit、QSVT angles、
  package versions、逻辑资源与编译后资源；
- 真实 counts parser 对缺失/空/非法 bitstring fail closed，并保存失败原因；
- 真机提交和结果取回均默认关闭。

## 尚未验证

- 未使用或验证任何 API token；
- 未查询实时后端状态；
- 未提交 Quafu-SQC 任务；
- 未验证远端编译器是否接受该具体 QASM；
- 未获得真机 counts、校准信息或 transpiled QASM；
- 未实现矩阵 block encoding、量子 ridge inverse、amplitude amplification
  或 classical shadows；
- 未做严格系统级在线 streaming；
- generic Aer 参数不是 Quafu-SQC 实时校准。

第一次真机运行前应先：

1. 从 `QuantumComputing` 环境启动 Jupyter；
2. 在 `09` 中先审计 flat-QOS manifest；
3. 查询实时后端状态；
4. 先选择一条 H-only sanity check；
5. 对同一低深度 exact-pair 或 balanced-stream 线路比较 `quarkcircuit` 与
   `qsteed`；
6. 保存 `tid`、raw/corrected counts 和完整 transpiled QASM，检查二比特门数、
   二比特 depth 与残留三比特门；
7. 再决定是否运行 flat IID $M$ sweep；
8. 只有 flat controls 可接受时，才考虑 `10` 的 deep $M=8$ full-QSVT verifier。
