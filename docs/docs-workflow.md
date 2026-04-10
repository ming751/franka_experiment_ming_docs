# 文档维护

## 安装依赖

```bash
python3 -m pip install -r requirements-docs.txt
```

当前站点使用：

- `mkdocs`
- `mkdocs-material`

## 本地预览

```bash
mkdocs serve
```

默认会启动一个本地开发服务器，边改边刷新。

## 静态构建

```bash
mkdocs build --strict
```

建议在提交前至少跑一次 `--strict`，这样能尽早发现：

- nav 路径错误
- 文件缺失
- 明显的文档结构问题

## 文档结构约定

当前文档站采用：

- `README.md`：仓库入口与最短使用说明
- `mkdocs.yml`：站点导航与主题配置
- `docs/*.md`：详细文档页面

推荐的维护原则：

- 把根 `README` 保持成入口页，而不是把所有信息继续堆回一个超长文件
- 把“经常变化的命令与映射关系”写到独立页面，避免埋在长文中
- 以代码和配置作为权威来源，文档只做解释与导航

## 建议的更新顺序

当你修改系统行为时，通常按下面顺序同步文档：

1. 先改代码 / launch / YAML / msg
2. 再更新对应说明页
3. 如果导航需要变化，再改 `mkdocs.yml`
4. 最后执行 `mkdocs build --strict`
