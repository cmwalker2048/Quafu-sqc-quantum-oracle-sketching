# IMDb $N=4$ Boolean-QOS 真机缩放实验

对应 notebook：

- `notebooks/13_imdb_qos_hardware_scaling.ipynb`

这是一项受论文 Figure 3(b) 启发的真机小规模实验，不是论文原图的严格复现。
它使用 Notebook 09 已生成的 IMDb 符号向量

$$
S_{\mathrm{IMDb}}=(+1,-1,-1,-1),
\qquad
f=(1-S_{\mathrm{IMDb}})/2=(0,1,1,1).
$$

论文记号中的 Boolean domain size 是 $N=4$；地址寄存器只需要

$$
n=\log_2N=2
$$

个量子比特。这个 Boolean-QOS 线路不使用 QSVT。

## 为什么单独建立 Notebook 13

Notebook 09/11 已经实现同一种 2-qubit QOS echo primitive，但 Notebook 11
同时包含位序校准、compiler A/B、旧 IMDb IID 任务和 full-QSVT，已有结果也混合了
不同 backend。Notebook 13 只负责本次缩放 campaign：

- 一次生成 $M=4,8,16,32,64,96$ 六条静态 QASM；
- 固定一个 backend、compiler 和 physical-qubit pair；
- 一次批量提交六条异步任务；
- 自动保存 task ID、raw result、returned QASM 和编译资源；
- 把论文的离线算子误差与真机 echo fidelity 分开画图。

Notebook 09、10、11 均不需要修改。

## $\epsilon$ 与 $F_{\rm echo}$

论文开源 benchmark 的纵轴为 expected-unitary operator error：

$$
\epsilon_{\mathrm{paper}}
=
\left\|\mathbb E[V_M]-O_f\right\|_{\mathrm{op}}.
$$

对于均匀 Boolean samples：

$$
\epsilon_{\mathrm{paper}}
=
\max_j
\left|
\left(
1-\frac1N+
\frac{e^{i\pi N f(j)/M}}N
\right)^M
-(-1)^{f(j)}
\right|.
$$

本项目真机直接测量：

$$
F_{\mathrm{echo}}
=
\left|
\langle +^{\otimes n}|O_f^\dagger V_M|+^{\otimes n}\rangle
\right|^2
=P(00).
$$

$\epsilon$ 是最坏坐标误差，$F_{\rm echo}$ 是一个输入态上的平均相位 overlap。
因此 `1 - F_echo` 只能叫 echo infidelity，不能标成论文的 $\epsilon$。

当前 $N=4$ 的离线 expected-unitary 参考值为：

| $M$ | $\epsilon_{\mathrm{paper}}$ | $\mathbb E[F_{\rm echo}]$ |
|---:|---:|---:|
| 4 | 1.0625 | 0.2266 |
| 8 | 0.8752 | 0.2997 |
| 16 | 0.6109 | 0.4264 |
| 32 | 0.3716 | 0.5942 |
| 64 | 0.2067 | 0.7497 |
| 96 | 0.1430 | 0.8199 |

## 运行

请在 VS Code 中选择 `QuantumComputing` kernel。

### 1. 离线检查

保持：

```python
ACTION = "dry_run"
CONFIRM_REAL_SUBMISSION = ""
```

运行全部单元。此阶段不需要 token，不会联网或提交任务。

### 2. 查询队列

```python
ACTION = "status"
```

运行全部单元。token 可由隐藏输入框输入，也可使用环境变量
`QPU_API_TOKEN`。

### 3. 一次提交六条

```python
ACTION = "submit"
CONFIRM_REAL_SUBMISSION = "RUN_QUAFU_HARDWARE"
```

默认：

```python
INCLUDE_H_ONLY_CONTROL = False
```

所以会一次提交六条 $M$ 线路。Quafu-SQC 的 `run(task)` 是异步接口：
notebook 会依次取得六个 task ID 并立即保存回执，不等待上一条执行完成。
如果提交中途因网络中断，再用同一 campaign 执行 `submit` 时，registry 中已经
成功登记的任务会安全跳过，只继续缺失的任务。

提交后立即清空确认字符串，避免重复提交：

```python
CONFIRM_REAL_SUBMISSION = ""
```

### 4. 查询本批任务状态

```python
ACTION = "status"
```

运行全部单元会显示当前 `CAMPAIGN_LABEL` 下 registry 里的任务状态。

### 5. 取回

```python
ACTION = "fetch"
CONFIRM_REAL_SUBMISSION = ""
```

运行全部单元。未完成任务会被跳过；稍后再次运行即可。无需逐个填写 task ID。

## 是否需要重跑校准

本实验只统计 `00`，bitstring 反转后仍为 `00`，因此不需要为这个指标重复
one-hot 位序校准。Notebook 11 的位序校准主要来自 Baihua，也不能严格视为
Shenglian 的同日校准。

建议保持：

- 同一 backend；
- 同一 compiler；
- 同一 `TARGET_QUBITS`；
- 六条任务在相近时间提交。

若希望增加当日浅线路健康基线，可设置：

```python
INCLUDE_H_ONLY_CONTROL = True
```

此时会提交七条任务。

## 解释限制

论文 Figure 3(b) 使用 $N=125,250,500,1000$、$M=10^5$ 到 $10^8$、
随机 Boolean functions、expected-unitary 无噪声模拟，并对每个点做 100 次重复。
本实验只有一个 IMDb target、$N=4$ 和每个 $M$ 一条固定 stream，因此：

- 可以称为 `IMDb-derived N=4 Boolean-QOS hardware scaling pilot`；
- 不能称为论文 Figure 3(b) 的严格复现；
- 不能用六个点验证 $M=\Theta(N/\epsilon)$；
- shots 的误差条只反映同一 QASM 的测量统计，不反映不同 IID streams 的波动；
- 若要估计 stream-to-stream error bar，每个 $M$ 至少需要若干独立 QASM。

静态线路的逻辑 CX 数大约随 $1.5M+6$ 增长，$M=96$ 已接近 150 个
双比特门。当前设备噪声很可能压过理论采样改善；这正是本真机 pilot 要观察的
sampling-error 与 circuit-noise 竞争。
