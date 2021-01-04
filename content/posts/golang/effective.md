---
title: "Effective Go"
date: 2020-12-24T13:59:29+08:00
draft: true
toc: false
images:
tags: 
  - golang
---
## Go的规范

### 命名规则

1. 包名应该简介明了而易于理解。
2.  包应当以小写的单个单词来命名，且不应使用下划线或驼峰记法 

### 获取器

1. 若你有个名为 `owner` （小写，未导出）的字段，其获取器应当名为 `Owner`（大写，可导出）而非 `GetOwner`

2. 若要提供设置器方法，`SetOwner` 是个不错的选择

```go
owner := obj.Owner()
if owner != user {
    obj.SetOwner(user)
}
```



### 接口命名

1. 只包含一个方法的接口应当以该方法的名称加上 - er 后缀来命名，如 `Reader`、`Writer`、 `Formatter`、`CloseNotifier` 等。

### 驼峰命名

1. Go 中的约定是使用 `MixedCaps` 或 `mixedCaps`  而不是下划线来编写多个单词组成的命名。 