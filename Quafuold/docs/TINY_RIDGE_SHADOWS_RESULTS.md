# Tiny Ridge + inverse QSVT + classical shadows 结果

验证日期：2026-07-29  
入口：[`../notebooks/12_tiny_ridge_qsvt_shadows.ipynb`](../notebooks/12_tiny_ridge_qsvt_shadows.ipynb)  
真机步骤：[`../TINY_RIDGE_SHADOWS_README.md`](../TINY_RIDGE_SHADOWS_README.md)

## 实验链

$$
\text{IMDb train}
\rightarrow
z(x)\in\mathbb R^2
\rightarrow
w=(X^\top X/N+\lambda I)^{-1}X^\top y/N
\rightarrow
\text{inverse QSVT}
\rightarrow
|\widetilde w\rangle
\rightarrow
\text{global-Clifford shadow}.
$$

监督式词汇集合、标准化参数、Ridge matrix 和 test feature 都是经典状态。量子
线路只执行给定的 2×2 matrix/rhs 和 shadow readout；它没有实现 scalable QOS
data loading。

## IMDb 与经典基线

模型选择严格限制在 official train：

1. train 内部拆分 fit/validation；
2. fit 学词方向；
3. validation 从 $K=16,32,64,128,256$ 选择每个极性的词数；
4. 固定 $K=256$ 后在完整 train 重学词表和 Ridge；
5. official 25,000 test 只作最终评估。

| 方法 | official-test accuracy |
|---|---:|
| polarity rule $w\propto(1,-1)$ | 0.84876 |
| exact 2D Ridge | 0.84848 |
| 12-setting ideal QSVT-shadow | 约 0.84900 |

polarity rule 略高于 exact Ridge，所以 accuracy 对 solver 几乎没有辨别力。
本实验验证 QSVT 的主要证据是 heralding 和 target-inverse verifier，不是分类
提升。

Ridge matrix 的特征值为约 0.65540 和 1.44460。正规方程条件数为 2.20413；
论文以 singular value 表达时，对应
$\kappa_{\rm reg}\approx\sqrt{2.20413}=1.48463$。

## inverse QSVT

将 $A$ 除以最大特征值，使用

$$
p(x)=a_1x+a_3x^3
$$

在两个已知正谱点精确插值 $c/x$，并在 $[-1,1]$ 上把
$|p(x)|$ 限制到 0.8。`pyqsp` 生成 symmetric-QSP phases；额外 real ancilla
实现 $(V+V^\dagger)/2$。

| 指标 | 理想值 |
|---|---:|
| QSVT heralding probability | 0.104569 |
| postselected target-state fidelity | 约 1.0 |
| interferometric heralding probability | 0.189339 |
| interferometric target-state fidelity | 约 1.0 |

这个三次多项式只在当前两个谱点精确，不是连续谱区间上的通用 inverse
approximation。当前还只做 postselection，没有 amplitude amplification。

## interferometric classical shadow

普通 $|w\rangle$ 的投影概率会丢失分类符号。这里准备

$$
|\widetilde w\rangle
=\frac{|0\rangle|0\rangle+|1\rangle|w\rangle}{\sqrt2},
$$

并对每个 two-qubit global Clifford 的结果使用

$$
\widehat\rho=5C^\dagger|b\rangle\langle b|C-I.
$$

Quafu 一条任务不能逐 shot 更换 Clifford，所以当前方案是 12 个 distinct static
settings，每个 2,048 shots。重复 shots 只降低同一 setting 内的 measurement
noise，不能当作 24,576 个独立 Cliffords。

本地 ideal Aer 的完整 setting 集结果：

| 线路族 | weight norm ratio | signed cosine | signed-overlap RMSE | test accuracy |
|---|---:|---:|---:|---:|
| shallow shadow control | 约 0.620 | 约 0.99993 | 约 0.305 | 约 0.84872 |
| full QSVT-shadow | 约 0.623 | 约 0.99943 | 约 0.303 | 约 0.84900 |

因此 12 settings 很好地恢复了轴和符号，但没有准确恢复 overlap 幅值。它应称为
sign/classification pilot。notebook 对 settings 做 cluster bootstrap；该区间是
固定 test set 上的探索性稳定性度量，不是论文 i.i.d.-Clifford +
median-of-means 的严格置信保证。

后处理显式重建完整 $4\times4$ density estimator，只适合这个 $D=2$ 自检。
可扩展版本应直接估计需要的 observables，而不是存整个 density matrix。

## 本地编译资源

QASM basis 为 OpenQASM 2.0 的 `u1/u2/u3/cx/measure/barrier`：

| 阶段 | QASM 数 | compiled depth | compiled CX |
|---|---:|---:|---:|
| q0–q3 calibration | 4 | 2 | 0 |
| direct Ridge echo control | 1 | 3 | 0 |
| QSVT prepare/verify | 2 | 32 | 18 |
| shallow shadow control | 12 | 5–9 | 2–4 |
| full QSVT-shadow | 12 | 181–185 | 95–97 |

full QSVT-shadow 使用 exact tiny-unitary dense resynthesis。它把当前 4-qubit
线路压到可尝试的深度，但合成成本随 qubit 数指数增长，不是 scalable
sparse-QSVT 编译策略。远端路由资源必须以 returned QASM 为准。

## 真机证据要求

完整硬件结论至少需要：

- 显式冻结同 backend/compiler/固定 physical layout 的 task cohort，并人工核对
  平台时间窗；
- q0–q3 calibration 每条恰好一个可信结果，主峰正确且 expected-state Wilson
  95% 下界至少 0.50；
- QSVT verifier 每条恰好一个可信结果，heralding Wilson 下界至少 0.02，
  conditional-fidelity Wilson 下界大于 0.50；
- 同一 cohort 的 12-setting shallow shadow control 完整且无重复，heralding、
  signed cosine、signed-overlap RMSE 和 accuracy 通过 notebook 中的保守门槛；
- full QSVT-shadow 使用相同 Clifford seeds；
- receipt 中的 QASM/Clifford/model/data hashes 与取回分析完全一致；
- 同时报出 accepted events、setting 数、returned CX/depth、signed cosine、
  overlap error 和 accuracy。

这些阈值是实验发布门槛，不是理论优势证明。最终 cohort 应包含 4 条
calibration、1 条 verifier、12 条 shallow control 和 12 条 full shadow，共
29 个明确 task id。少于 12 个 full settings、存在重复或任何前置门槛失败时，
真机点只能标为 provisional。当前项目尚未提交任何 Quafu-SQC 任务，所以这里
所有量子数值仍是本地理想模拟，不是真机结果。

## 与论文 Figure 2 的关系

论文 Figure 2 的三条曲线共享经典分类 accuracy，只改变机器空间记账；原 IMDb
脚本使用 min-df 截断和五折 CV。本项目左面板改用 feature hashing 和 official
held-out split，只是 paper-inspired re-accounting。

4 个物理 qubits 不包含监督式词汇投影、matrix/rhs 和线路描述的经典存储，不能
与 Figure-2 machine-size account 使用同一纵轴。最终图因此把它们放在两个独立
面板：

![Paper-inspired accounting and tiny sign pilot](../assets/figures/tiny-ridge-machine-size-comparison.png)
