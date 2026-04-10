# Franka Experiment Documentation

这个仓库用于公开维护 `franka_experiment_ming` 项目的文档部分，适合在毕业论文、项目主页或评审材料中直接引用。

当前公开内容主要包括：

- 系统架构说明
- 快速开始与运行命令
- 日志与观测说明
- 闭环实验后的绘图与后处理流程

说明：

- 当前文档仓库与主源码仓库分离维护
- 主源码仓库可以继续保持私有
- 文档内容来自主仓库中的 `docs/`，可按需同步更新

## 文档入口

- [文档首页](docs/index.md)
- [快速开始](docs/quickstart.md)
- [架构总览](docs/architecture.md)
- [模块说明](docs/packages.md)
- [运行与命令](docs/commands.md)
- [绘图与后处理](docs/plotting.md)
- [观测与日志](docs/observability.md)

## 本地预览

安装依赖：

```bash
python3 -m pip install -r requirements-docs.txt
```

本地预览：

```bash
mkdocs serve
```

静态构建：

```bash
mkdocs build --strict
```

## 发布教程

1. 在 GitHub 上创建一个新的公开仓库，例如：

```text
franka_experiment_ming_docs
```

2. 在当前目录初始化并提交：

```bash
cd /workspace/franka_experiment_ming_docs
git init
git checkout -b main
git add .
git commit -m "Initial public documentation repository"
```

3. 添加远端并推送：

```bash
git remote add origin git@github.com:ming751/franka_experiment_ming_docs.git
git push -u origin main
```

4. 在毕业论文中引用文档入口：

```text
https://github.com/ming751/franka_experiment_ming_docs
```

或者直接引用文档目录：

```text
https://github.com/ming751/franka_experiment_ming_docs/tree/main/docs
```

## 后续同步方法

如果主仓库文档有更新，可重新同步：

```bash
rm -rf /workspace/franka_experiment_ming_docs/docs
mkdir -p /workspace/franka_experiment_ming_docs/docs
cp -r /workspace/docs/. /workspace/franka_experiment_ming_docs/docs/
```

然后在文档仓库里提交并推送：

```bash
cd /workspace/franka_experiment_ming_docs
git add .
git commit -m "Sync docs from main workspace"
git push
```
