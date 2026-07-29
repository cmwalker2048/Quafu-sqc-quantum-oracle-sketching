# Circuit diagrams

这些 PNG 由 notebooks 中的 Qiskit 电路对象或实际 OpenQASM 2.0 自动生成：

- `bell-circuit.png`：环境 smoke test；
- `projector-phase-update.png`：一个经典样本对应的 projector-phase 增量门；
- `qos-verification-overview.png`：完整验证流程的逻辑概览；
- `qos-full-n2.png`：由待提交 QASM 解析得到的逐门线路；
- `hadamard-overlap-test.png`：centroid 分类的重叠读出。
- `qos-equivalence-n1.png`：`05` 中用于等价测试的实际 echo QASM；
- `projector-phase-update-n3.png`：$n=3$ 的双工作 ancilla projector update；
- `qos-campaign-balanced-n1.png`：`06` 中保留样本 barrier 的 balanced control。

不要手工编辑这些图片。修改线路后重新运行相应 notebook，以保持图片、QASM
和代码一致。

在 `projector-phase-update-n3.png` 中，`q_0`–`q_2` 是 address，`q_3` 是中间
work，`q_4` 是 phase flag；目标地址 5 的二进制为 $101$。Qiskit 使用
little-endian 位序，所以只对中间位 `q_1` 加 X mask。barrier 是编译/显示边界，
不是物理量子门。
