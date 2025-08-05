## 贡献指南 (CONTRIBUTING.md)

本文档参考了 [intro2oss][1] 项目规范，并结合了 learning-qemu-docs 项目的实际需求。

## 本地环境配置

为了能够顺利地贡献内容并验证您的修改，请先完成以下准备工作：

1. 克隆仓库并配置上游 (remote upstream)

```bash
# 首先 Fork 本仓库到您的 GitHub 账户，然后执行以下操作：
git clone https://github.com/<你的 GitHub 用户名>/learning-qemu-docs.git
cd intro2oss

# 添加上游仓库以便后续同步更新
git remote add upstream https://github.com/gevico/learning-qemu-docs.git

# 注意：更推荐使用 ssh 的方式克隆,具体可参考 github 官方文档
```

2. 安装依赖

```bash
# 安装 Python 依赖
pip install -r requirements.txt

# 安装前端工具（如 markdownlint 等）
npm install
```

3. 本地预览文档

```bash
mkdocs serve
```

然后在浏览器中打开 http://127.0.0.1:8000 进行预览。

4. 常用 Git 命令备忘

```bash
# 查看当前状态
git status

# 基于 main 创建工作分支
git checkout -b feature/your-feature-name

# 拉取上游最新代码（在 main 分支）
git checkout main
git pull upstream main
```

## 书写规范

1. 从标题开始书写

文档的 一级标题已经在`docs/index` 中定义，你可以选择从当前文档重新定义一级标题，这样会覆盖原来的一级标题（如果有需要）。

推荐从二级标题开始

示例：

```
## 这是一个文档的章节名称
```

2. 主要作者署名（可选）

如果您编写了某个章节的主要内容，请在章节的开头添加主要作者提示框。

格式如下，不同作者名字之间用中文全角顿号（「、」）分隔：

```
!!! note "主要作者"

    [@example][github.com/example]、[@new-contributor][github.com/new-contributor]
```
3. 为图片添加配字

所有图片都应在图片下方添加配字说明，并在配字的下一行写上 {: .caption } 以应用特定样式。

格式如下：

```
![图片替代文本](images/example-image.png)

图 1. 这是一张示例图片的配字说明
{: .caption }
```

请确保配字简洁明了，并对图片内容进行必要的解释。图号建议按章节顺序编号（如图 1.1, 图 1.2, 图 2.1 等）或按文档顺序编号。

4. 使用提示框 (Admonition)

提示框（Admonition）是 mkdocs-material 主题的特色功能，能够有效突出显示特定信息。建议根据内容性质选用以下类型的提示框：

- note：普通提示，用于补充说明或引人注意的信息

```
!!! note "请注意"

    这是一条普通提示信息。
```

- warning：警告信息，需要读者特别注意，否则可能出现非预期行为

```
!!! warning "警告"

    进行此操作前请务必备份数据。
```

- danger：危险操作提示，可能导致严重后果（如数据丢失、系统损坏等）

```
!!! danger "危险操作"

    以下命令将删除所有数据，请谨慎操作！
```

- tip：小技巧或建议，帮助读者更高效地学习或操作

```
!!! tip "小技巧"

    您可以使用快捷键 `Ctrl + S` 保存文件。
```

- example：用于展示示例代码、配置或操作步骤

```markdown
!!! example "示例：Python 'Hello World'"

    ```python
    print("Hello, World!")
    ```
```

- question：提出问题，引导读者思考

```
!!! question "思考题"

    为什么我们需要使用版本控制系统？
```
**可折叠的提示框：**

您可以创建可折叠的提示框。

默认折叠：使用三个问号 ???。用户需要点击标题才能展开内容

```
??? note "这是一个默认折叠的提示框 (点击展开)"

    这里是折叠的内容。
```

默认展开：使用三个问号加一个加号 ???+。内容默认是可见的，但用户可以点击标题将其折叠

```
???+ tip "这是一个默认展开的可折叠提示框 (点击折叠)"

    这里是默认展开的内容。
```

这种方式可以帮助您更好地组织较长的补充信息或示例，保持文档的整洁性，同时允许读者按需查看详细内容。

## Commit 规范

整体参考 QEMU 的 Commit 风格，为了保持 Commit 历史的清晰和可追溯性，请按照一下格式提交：

```bash
scope: subject
```

常见的 scope 有:

- repo: git 相关或者版本控制

- docs: 文档相关

- 仓库相对路径，比如 docs/ch1

## Pull Request (PR) 规范

在创建 Pull Request 时，请确保遵循以下步骤并填写提供的 PR 模板：

1. PR 标题：清晰概括 PR 的主要内容，例如 Fix: Typo in Chapter 1 Introduction 或 Docs: Add new section on OSS licenses

2. PR 内容：我们提供了 PR 模板，请认真对照内容进行填写

PR 模板示例：

```
## 关联章节/文件
<!-- 请填写您修改的文档路径或代码文件路径，例如： -->
- 文档路径：`docs/1-install/2-partition.md`

## 修改类型
<!-- 请选择一项或多项，并用 [x] 标记 -->
- [ ] 内容补充
- [ ] 错别字/语句修正
- [ ] 格式调整
- [ ] 图片/附件更新
- [ ] 代码示例修改
- [ ] 其他 (请说明):

## 修改描述
<!-- 请简要描述您本次修改的主要内容和原因 -->
-
-

## 本地验证
<!-- 请确保您已在本地完成以下检查 -->
- [ ] 已通过 `autocorrect .` (或 `autocorrect --lint .` 检查并通过)
- [ ] 已执行 `mkdocs serve` 并在本地预览，确认页面渲染正常、链接有效、排版美观

## 附加说明 (可选)
<!-- 如有其他需要说明的事项，请在此填写 -->
-
```
