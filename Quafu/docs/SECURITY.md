# 凭据与数据安全

## 立即行动

任何曾经出现在聊天消息、notebook 源码、截图或 Git commit 中的 API token，
都应视为已经泄露。请在 Quafu/SQCLab 平台撤销旧 token 并生成新 token。

本项目没有复制或使用用户提供过的明文 token，也不会用已暴露 token 提交任务。

## 推荐方式

启动 Jupyter 前，在当前终端临时设置环境变量：

```bash
export QPU_API_TOKEN='新生成的令牌'
/opt/miniconda3/envs/QuantumComputing/bin/jupyter lab ~/Desktop/Quafu
```

环境变量只对当前 shell 及其子进程有效。关闭终端后可重新设置。

也可以让 notebook 使用 `getpass.getpass()` 隐藏输入。不要执行：

```python
token = "明文令牌"
```

## Git 检查清单

推送前确认：

- `.env`、`.env.*`、凭据文件已被忽略；
- notebook 单元源码和输出中没有 token；
- shell 历史、截图和导出的 HTML 中没有 token；
- `git diff --cached` 已人工检查；
- 曾经提交过的 token 已撤销，而不是仅删除当前文件。

## notebook 注意点

即使隐藏输入没有显示在单元里，也不要把 `token` 变量打印出来。异常信息、
调试日志和 HTTP 请求头也可能泄露凭据。

云端结果可以保存 task id 和实验元数据，但不应保存认证头或完整请求对象。

