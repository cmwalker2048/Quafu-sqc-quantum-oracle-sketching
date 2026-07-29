# Research figures

这些 PNG 由 notebooks 的数值结果自动生成：

- `expected-unitary-validation.png`：expected-operator surrogate、真实解析平均与
  显式随机流的区别；
- `qasm-equivalence-validation.png`：直接相位公式与 QASM 中提取的
  $F_{\rm echo}$；完整 statevector 误差另在 `05` 表中给出；
- `qos-gate-pressure.png`：只比较 IID 系列内部的 CX proxy 增长；60/180 是
  项目筛选启发式，不是 Quafu-SQC 设备上限；
- `hardware-aware-qos-tradeoff.png`：采样收益、generic gate noise 与门数竞争。
- `tiny-ridge-qsvt-polynomial.png`：只在 2×2 matrix 两个谱点精确的 bounded
  degree-3 inverse polynomial；
- `tiny-ridge-shadow-accuracy.png`：polarity rule、exact tiny Ridge/QSVT 与
  shadow sign/classification pilot；
- `tiny-ridge-machine-size-comparison.png`：左面板是 paper-inspired
  machine-size account，右面板是独立资源范围的 tiny mechanism，禁止把两者
  当作同一纵轴优势比较。

不要手工修改。重新运行对应 notebook 以保持图片、数据和代码一致。
