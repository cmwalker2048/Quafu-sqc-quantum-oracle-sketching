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

`12_tiny_ridge_qsvt_shadows.ipynb` 的
`tiny_ridge_qsvt_shadows/` 保存 2D IMDb Ridge、inverse QSVT 和
interferometric global-Clifford shadow：

- 无原始评论文本的派生 2D train/test cache 与 hash；
- normal-equation block encoding、spectrum-matched degree-3 polynomial、
  QSVT phases、条件数和理想 postselection 指标；
- 4 条 q0–q3 one-hot calibration、3 条 Ridge、12 条 shallow shadow control
  与 12 条 full QSVT-shadow，共 31 条 4q QASM；
- QASM hash/schema/basis audit 和理想 Aer counts；
- shadow weight norm、signed direction cosine、signed-overlap
  MAE/RMSE/max error、classification accuracy 与 distinct-setting cluster
  bootstrap 探索性区间；
- paper-inspired machine-size account 和 tiny mechanism 的分面图。

`tiny_ridge_qsvt_shadows/raw_results/`、`task_registry.json` 和
`hardware_results.json/csv` 为真实设备产物，默认忽略。receipt 会冻结完整
manifest entry、Clifford matrix/seed、QASM/model/data hashes；取回时 provenance
不一致的 counts 只保存 raw payload，不进入分析。

`11_quafu_hardware_validation.ipynb` 使用
`hardware_validation/` 作为统一真机工作目录：

- `offline_audit.json/csv`：73 条 flat/general/calibration QASM 的离线审计；
- `audited_run_plan.csv`：冻结后的候选运行计划；
- `calibration_qasm/`：2q/4q one-hot bit-order controls；
- `task_registry.json`：提交 task id ledger；
- `hardware_results.json/csv`：统一解析结果；
- `raw_results/`：status、receipt、raw payload、returned QASM 与真机图。

后三类真实云端产物默认被忽略，发布前需人工审核。

`13_imdb_qos_hardware_scaling.ipynb` 使用
`qos_hardware_scaling/` 保存独立的 IMDb $N=4$ Boolean-QOS 缩放 campaign：

- $M=4,8,16,32,64,96$ 六条 2q 静态 QASM；
- 固定 target、nested IID stream、QASM hash 和逻辑门资源的 manifest；
- 论文 expected-unitary operator error、固定 stream operator error、
  理想 $F_{\rm echo}$ 与解析 $\mathbb E[F_{\rm echo}]$；
- 离线审计 CSV/JSON 和 $\epsilon$/$F_{\rm echo}$/CX 分面图；
- 可选的 task registry、raw result、returned QASM 与硬件结果 CSV/JSON。

真实云端 registry 与 raw results 在发布前仍需人工审核；图中不会把
$1-F_{\rm echo}$ 冒充论文 Figure 3(b) 的 operator error $\epsilon$。
