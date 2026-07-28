# Quafu 工具链选型

## 结论

第一阶段采用：

```text
NumPy/Matplotlib 离线验证
        +
手写、可审计的 OpenQASM 2.0 生成器
        +
Qiskit 2.5 解析与线路可视化
        +
quafusqc 3.3.9 的 quafu.Task 云端接口
```

暂不安装 QuarkStudio，也不使用 PyQuafu。

PyQuafu 尚未被上游正式标记为 deprecated，但它对应旧的云端通路；新的
Quafu-SQC 文档已经采用 `quafusqc`。此外 `pyquafu` 和 `quafusqc` 都写入
顶层 `quafu` 包，放在同一环境会产生 namespace 覆盖风险。

## 本机实测环境

`quafusqc 3.3.9` 只安装在：

```text
/opt/miniconda3/envs/QuantumComputing
Python 3.13.7
```

默认 `python3` 是系统 Python 3.9，默认 Miniconda base 也没有 `quafusqc`。
因此必须从 `QuantumComputing` 环境启动 Jupyter。

截至 2026-07-28 的工具定位：

| 工具 | 当前状态 | 本项目结论 |
|---|---|---|
| `quafusqc 3.3.9` | 已安装；轻量云任务客户端 | 第一阶段采用 |
| `qiskit 2.5.1` | 已安装；QASM 解析和线路绘图 | 采用，但不负责 Quafu 提交 |
| `quarkstudio 7.3.9` | 未安装；完整 QOS/实验室服务栈 | 当前过重，不作为核心依赖 |
| `quarkcircuit 0.5.13` | 未安装；线路构造与转译 | 需要复杂线路对象时再评估 |
| `pyquafu 0.4.5` | 未安装；旧云通路兼本地模拟 | 不与 `quafusqc` 共装 |

发行包与 import 名不同：

```text
pip 包名: quafusqc
Python import: quafu
```

主要 API：

```python
from quafu import Task

tmgr = Task(token)
tmgr.status()
tmgr.run(task)
tmgr.status(tid)
tmgr.result(tid, timeout=...)
```

`Task(token)` 会立即联网验证 token，所以离线 notebook 不应为了“测试导入”
而实例化它。

## 为什么当前不用 QuarkStudio

QuarkStudio 官方文档同样支持 OpenQASM 2.0 和云端任务管理，并提供
`from quark import Task`。但当前本机没有安装 `quarkstudio`、`quark` 或
`quarkcircuit`。

Boolean QOS 第一阶段只需要：

- `h`
- `x`
- `cx`
- `ccx`
- `u1`
- measurement

这些都能明确写成 OpenQASM 2.0。直接生成 QASM 有三个优点：

1. 不增加依赖；
2. 每个样本对应的门可以逐行审计；
3. 不把 SDK 线路对象误当作 QOS 理论的一部分。

QuarkStudio 的 QuarkCircuit 当前也不提供本地量子模拟，因此安装它不能替代
本项目的 NumPy/密度矩阵验证层。

后续出现以下需求时再评估安装 QuarkStudio：

- 本地绘制后端耦合图；
- 更高阶 MCX 的可靠分解；
- 复杂线路对象和编译 pass；
- 脉冲或设备工作流；
- 需要读取 `Task.backend()` 的可视化结果。

## 当前 quafusqc 的已知注意点

- `Task.backend()` 本地路径依赖 `quark.circuit.Backend`；当前未安装
  QuarkStudio 时不要调用。
- `Task` 构造会立即验证凭据，假 token 不能用于 dry run。
- 云任务默认异步；必须保留 `tid` 并轮询终态后再取结果。
- `shots` 应按官方后端要求设置，示例使用 1024。
- 远端编译器选项可以写为 `quarkcircuit`；这不等于本地已经安装该包。
- 后端状态和排队长度是实时信息，旧 notebook 输出不能复用。

## Qiskit 在本项目中的边界

Qiskit 只用于：

- `qiskit.qasm2.loads(...)` 独立解析生成的 OpenQASM 2.0；
- `QuantumCircuit.draw(output="mpl")` 在 notebook 中生成线路图；
- 在提交前检查 qubit/clbit 数量和逻辑门计数。

它不保存 Quafu token、不提交 Quafu 任务，也不替代 `quafusqc`。当前没有安装
Qiskit Aer，因为本项目已有明确的 NumPy/密度矩阵验证层，线路绘图不需要 Aer。

## 任务模板

```python
task = {
    "chip": "Baihua",
    "name": "qos_boolean_n2",
    "circuit": qasm_text,
    "shots": 1024,
    "compile": True,
    "options": {
        "compiler": "quarkcircuit",
        "correct": False,
        "open_dd": None,
        "target_qubits": [],
    },
}
```

提交动作在 notebook 中默认关闭，只有在检查 QASM、刷新 token、查看当前
后端状态后才应开启。

## 官方资料

- https://quafu-sqc.readthedocs.io/en/latest/
- https://quarkstudio.readthedocs.io/en/latest/
