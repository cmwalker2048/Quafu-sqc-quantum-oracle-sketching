# Data

此目录用于可公开、可复现的数据说明与本地缓存。

`08_imdb_general_vector_classification.ipynb` 和
`09_quafu_imdb_vector_pilot.ipynb` 使用 Stanford Large Movie Review Dataset
的官方 25,000 train / 25,000 test split。原始 `aclImdb` 只应放入
`data/cache/`、`data/raw/` 或由 `IMDB_DATA_ROOT` 指定的外部目录；这些本地
缓存不进入 Git。

推荐：

```bash
export IMDB_DATA_ROOT=/absolute/path/to/aclImdb
```

`08` 也带有默认关闭的官方下载 helper。显式开启后会下载
`aclImdb_v1.tar.gz`、验证 SHA-256
`c40f74a18d3b61f90feba1e17730e0d38e8b97c05fde7008942e91923d1658fe`
并做安全解压。仓库只保存下载地址、校验值、split 计数、实验参数和聚合结果。

IMDb 数据卡未给出常见开源许可证；未来 GitHub 仓库不应提交评论正文，只提交
下载与校验代码、split ID、聚合指标和不含原文的可复现元数据。
不要把私有数据、认证信息或大规模原始数据直接提交到 Git。
