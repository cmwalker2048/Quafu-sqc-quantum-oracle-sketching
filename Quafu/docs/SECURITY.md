# 凭据与数据安全

## 使用原则

本项目不在源码、notebook 输出、截图或 Git commit 中保存 API token。
真机单元只从临时环境变量或隐藏交互输入读取凭据。

## 推荐方式

最安全、也最适合初学者的方式，是保留 notebook 中的 `getpass.getpass()`，
每次运行时隐藏输入，不把 token 写进源码、环境文件或 shell history。

若确实要使用当前终端的临时环境变量，不要把真实 token 直接写在命令行里；
使用隐藏读取：

```bash
read -s QPU_API_TOKEN
export QPU_API_TOKEN
/opt/miniconda3/envs/QuantumComputing/bin/jupyter lab ~/Desktop/Quafu
```

输入 token 后按回车；内容不会回显，也不会作为命令文本进入常见 shell history。
环境变量只对当前 shell 及其子进程有效。关闭终端后可重新设置。不要执行：

```python
token = "明文令牌"
```

## Git 检查清单

推送前确认：

- `.env`、`.env.*`、凭据文件已被忽略；
- notebook 单元源码和输出中没有 token；
- shell 历史、截图和导出的 HTML 中没有 token；
- `git diff --cached` 已人工检查；
- notebook 输出和待提交 diff 已人工检查。

## notebook 注意点

即使隐藏输入没有显示在单元里，也不要把 `token` 变量打印出来。异常信息、
调试日志和 HTTP 请求头也可能泄露凭据。

云端结果可以保存 task id 和实验元数据，但不应保存认证头或完整请求对象。
