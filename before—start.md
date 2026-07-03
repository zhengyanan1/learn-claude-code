# 启动前准备

本项目建议使用 Python 虚拟环境运行示例脚本，避免把依赖安装到系统 Python 里。

## 首次准备

```sh
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

如果运行脚本时报环境变量缺失，再检查本地 `.env` 是否已经配置了对应变量，例如 `ANTHROPIC_API_KEY`。

## 日常启动

每次打开新的终端后，先进入项目目录并激活虚拟环境：

```sh
cd /Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code
source .venv/bin/activate
```

激活后可以运行示例脚本：

```sh
python s09_memory/code.py
```

如果没有激活虚拟环境，也可以直接用系统里的 Python 3：

```sh
python3 s09_memory/code.py
```

## 退出虚拟环境

```sh
deactivate
```
