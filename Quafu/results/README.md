# Results

此目录用于本地实验产物，例如：

- Quafu-SQC task metadata；
- 原始和编译后 QASM；
- counts；
- 汇总 CSV；
- 图像。

当前离线生成的 manifest、QASM、CSV 和配置 JSON 体积很小，并且构成 Notebook
结论的可复核证据，因此不再被 `.gitignore` 排除，后续可以随 GitHub 仓库提交。
真正云端返回的 `raw_results/` 与 `parsed_real_results.csv` 默认仍被忽略；发布前
应先脱敏、补齐设备/编译元数据，再有选择地纳入。

`03_quafu_boolean_qos_qasm.ipynb` 生成 `boolean_qos_n2.qasm` 与配套 metadata；
`05` 生成 expected-operator/完整 statevector 验证表和摘要。

`06_quafu_experiment_bundle.ipynb` 会生成：

```text
results/quafu_bundle/
├── manifest.json
├── manifest.csv
├── qasm/
├── raw_results/
├── generic_noise_config.json
└── generic_noise_*.csv
```

manifest 保存完整 stream、truth table、hash、`R_streams`、`S_shots`、逻辑门数、
配对控制 id 和任务模板。两个 no-barrier optimized twin 复用各自配对的
balanced stream，因而能把差异归因到 barrier/编译处理，而不是重新抽到的数据。
`raw_results/` 只在用户明确开启结果获取后写入真实返回；当前项目没有真机结果。

`07_general_vector_qos.ipynb` 生成：

- `general_vector_flat_validation.csv`；
- `general_vector_expected_scaling.csv`；
- `general_vector_active_raw.csv` 与 `general_vector_active_summary.csv`；
- `general_vector_reference_regression.json`；
- `general_vector_source_audit.json`。

这些文件分开记录 fetched/used 样本、active heralding probability、条件 fidelity、
pooled fidelity、bootstrap 区间、padding leakage、QSVT 拟合域和固定源码的
double-norm / unused-group 审计。

`08_imdb_general_vector_classification.ipynb` 生成官方 train/test 无泄漏的
IMDb 分类与 learned-weight QOS 数组结果：

- `imdb_dataset_manifest.json`；
- `imdb_hashing_ridge_accuracy.csv`；
- `imdb_general_vector_qos_runs.csv`；
- `imdb_general_vector_qos_sweep.csv`；
- `imdb_general_vector_summary.json`。

`09_quafu_imdb_vector_pilot.ipynb` 的
`imdb_vector_pilot/` 保存 IMDb-derived $D=4$ flat sign-vector、65 份可解析
QASM、完整 manifest 与离线 statevector 结果。真机模式会在提交前复核
QASM hash 与 2q/2c/2-measure schema，并把 receipt、raw result、returned
transpiled QASM/hash 和解析后的 `F_echo` 保存到 `raw_results/` 与
`parsed_real_results.csv`；这两类真机返回默认被忽略。

`10_quafu_general_vector_qsvt_pilot.ipynb` 的
`general_vector_qsvt_pilot/` 保存 4-qubit full general-vector
degree-4 QSVT：

- NumPy-vs-gate 逐振幅等价 JSON；
- $M=8,16$ 的 prepare/verify QASM；
- 编译门数、QASM hash、ideal Aer counts 和任务 manifest；
- 默认关闭的真机提交与结果取回模板。

`general_vector_qsvt_pilot/raw_results/` 和 `parsed_real_results.csv` 仍被忽略，
只有取得并审核真实设备返回后才应选择性发布。
