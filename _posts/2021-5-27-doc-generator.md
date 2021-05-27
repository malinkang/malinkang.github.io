---
title: 文档生成工具
date: 2021-01-30 15:12:35
draft: true
tags:
---

## MkDocs

* [MkDocs](https://www.mkdocs.org/)
* [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/getting-started/)

```
pip install mkdocs #安装
mkdocs --version #查看版本号
pip3 install mkdocs-material #安装主题
mkdocs new docs #创建
mkdocs serve
```

配置：

```
site_name: tpln
theme:
  name: material
  features:
    - navigation.sections
    - navigation.tabs
nav:
  - Home: index.md
```

