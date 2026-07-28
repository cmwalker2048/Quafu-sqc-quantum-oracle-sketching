# 本地验证记录

验证日期：2026-07-28  
验证环境：`/opt/miniconda3/envs/QuantumComputing/bin/python`  
Python：3.13.7  
quafusqc：3.3.9
Qiskit：2.5.1

## 执行方法

五个 notebook 均通过该环境的 Jupyter kernel 从头执行，并保留输出。
所有联网和真机提交开关保持 `False`。

Markdown 与 notebook Markdown 单元的数学定界符统一为：

- 行内公式：`$...$`
- 独立公式：`$$...$$`

## 结果

| Notebook | 结果 |
|---|---|
| `00_environment_and_quafu_access` | 环境/import/QASM dry run 通过；无网络请求 |
| `01_boolean_qos_core` | 通过；`M=8→1024` 时平均 `1-F_+` 从约 0.740 降到 0.032 |
| `02_sampling_and_noise_benchmark` | 通过；简化 phase-flip 模型出现经验 `M_opt=256` |
| `03_quafu_boolean_qos_qasm` | 通过；生成 3-qubit QASM，平衡 smoke test 的理想 `F_+=1` |
| `04_centroid_readout_bridge` | 通过；理想重叠分类 accuracy 约 0.877 |

所有 notebook 的 error output 数量均为 0。

## 线路图产物

Qiskit 从实际 QASM 或对应的逻辑线路对象生成：

- `assets/circuits/bell-circuit.png`
- `assets/circuits/projector-phase-update.png`
- `assets/circuits/qos-verification-overview.png`
- `assets/circuits/qos-full-n2.png`
- `assets/circuits/hadamard-overlap-test.png`

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

## 尚未验证

- 未使用或验证任何 API token；
- 未查询实时后端状态；
- 未提交 Quafu 任务；
- 未验证远端编译器是否接受该具体 QASM；
- 未获得真机 counts、校准信息或 transpiled QASM；
- 未实现通用向量 QOS、QSVT、ridge 或 classical shadows；
- 未做严格系统级在线 streaming。

第一次真机运行前应先：

1. 撤销此前暴露的 token，生成新 token；
2. 从 `QuantumComputing` 环境启动 Jupyter；
3. 在 `03` 中查询实时后端状态；
4. 人工检查 QASM 和元数据；
5. 只提交一条平衡 smoke-test 线路；
6. 保存 `tid` 和完整结果，再决定是否运行随机数据流 sweep。
