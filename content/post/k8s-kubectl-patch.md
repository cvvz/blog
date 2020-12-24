---
title: "kubectl patch"
date: 2020-11-22T23:16:44+08:00
draft: false
comments: true
keywords: ["kubectl", "kubernetes"]
tags: ["kubernetes", "kubectl"]
toc: true
autoCollapseToc: false
---

`kubectl patch` 用来修改 Kubernetes API 对象的字段。可以通过 `--type` 参数指定三种不同类型的 patch 方式：

- `strategic`：strategic merge patch
- `merge`： json merge patch
- `json`： json patch

实际使用情况：

- strategic merge patch 用的比较少；大多使用 json merge patch 和 json patch
- json merge patch 和 json patch 的具体区别可以查看[这篇文章](https://erosb.github.io/post/json-patch-vs-merge-patch/)
- json patch 相比于 json merge patch 使用起来复杂一点，但使用方法更灵活，功能更强大，副作用更少。因此更推荐使用。

## strategic merge patch

这是默认的patch类型，strategic merge patch 在进行 patch 操作时，到底是进行**替换**还是进行**合并**，由 Kubernetes 源代码中字段标记中的 `patchStrategy` 键的值指定。

具体来说：

- 如果你对deployment的 `.spec.template.spec.containers` 字段进行 strategic merge patch，那么新的 containers 中的字段值会合并到原来的字段中去，因为 `PodSpec` 结构体的 `Containers` 字段的 `patchStrategy` 为 `merge`。
- 如果你对deployment的 `.spec.template.spec.tolerations` 字段进行 strategic merge patch，那么会用新的 tolerations 字段值将老的字段值直接替换。

## json merge patch

**有相同的字段就替换，没有相同的字段就合并**。这在语义上非常容易理解，但是有以下弊端：

- 键值无法被设置为 `null`，设置为 `null` 的字段会直接被 json merge patch 删除掉
- 操作数组非常吃力。如果你想添加或修改数组中的元素，必须在copy原来的数组，并在其基础上进行改动。因为**新的数组会覆盖原来的数组**。

特别是第二点，这导致只要是和数组相关的patch操作，最好使用json patch。

## json patch

json patch 的格式如下：

```json
[
    {
        "op" : "",
        "path" : "" ,
        "value" : ""
    }
]
```

即由操作、字段路径、新值组成。具体例子查看[这篇文章](https://erosb.github.io/post/json-patch-vs-merge-patch/)。可以看到这种操作方式非常灵活。

## json patch 转义字符

- "~"（波浪线）对应的是："~0"
- "/"（斜杠）对应的是："~1"

具体可以查看这个[issue](https://github.com/json-patch/json-patch-tests/issues/42)中的讨论。
