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

- 网页版文档（推荐）: <https://ming751.github.io/franka_experiment_ming_docs/>

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

目标效果：

- GitHub 仓库源码入口：<https://github.com/ming751/franka_experiment_ming_docs>
- GitHub Pages 网页入口：<https://ming751.github.io/franka_experiment_ming_docs/>

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

4. 在 GitHub 仓库页面打开：

```text
Settings -> Pages
```

把 `Source` 设置为：

```text
GitHub Actions
```

不要点下面自动生成 Jekyll 的 `Configure` 按钮，仓库里的 workflow 会自己构建 MkDocs 并发布。

5. 首次推送后，GitHub Actions 会自动发布网页。成功后可通过下面地址访问：

```text
https://ming751.github.io/franka_experiment_ming_docs/
```

6. 在毕业论文中引用文档入口：

```text
https://ming751.github.io/franka_experiment_ming_docs/
```

如果你还想同时给出源码形式的文档入口，也可以附上：

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
