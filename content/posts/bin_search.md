---
date: 2022-03-04T00:38:48+08:00
showToc: true
TocOpen: false

hidemeta: false
comments: false

canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: false
ShowReadingTime: false
ShowBreadCrumbs: true
ShowPostNavLinks: true
author: "robinx"

# TODO
# weight: 3 # 优先级
title: "二分求峰型数组最大值-为什么不要mid-1"
description: ""
tags: ["algo"]
categories: ["posts"]
draft: false
# weight: 1
cover:
    # image: "http://120.76.102.194/pub/imgs/<modify_here>.png" # image path/url
    # alt: "should have a png here" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page

---

### 原始问题：

求一个峰型数组的最大值。

例如：
[0, 1, 2, 3, 2, 1, 0]

答案应该是：3

### 思路
很显然，使用二分。
一开始写的一种写法会死循环，如下：

```go
func peakIndexInMountainArray(arr []int) int {
    l, r := 0, len(arr)-1
    var mid int
    for l < r {
        mid = l + (r-l)>>1
        if arr[mid-1] < arr[mid] {
            l = mid
        } else {
            r = mid-1
        }
    }
    return l
}
```

原因是有一个案例是：[0, 1, 0]
会导致 `mid` 指向 `1` 的时候，`l` 也收缩到了 `1` ，然后判断 `arr[mid-1] < arr[mid]` 中，`mid-1` 已经下溢出了为 `l-1` ，不在当前求值区间`([l,r])`内。
如此便会导致死循环，一直走入第一个 `if` 判断中。

如何解决呢？

我们知道 3/2 == 1 ,而 2/2 == 1。除法是向下取整的，这导致 `r-l` 为奇数时，计算出的 `mid` 更靠近 'l' 而不是 'r'，如此便会导致 `mid = l + (r-l)>>1` 时，`mid` 更容易靠近可能值的左边界，也就是等于 `l`。所以这里要避免 `mid-1`的写法，正确写法如下：

```go
func peakIndexInMountainArray(arr []int) int {
    l, r := 0, len(arr)-1
    var mid int
    for l < r {
        mid = l + (r-l)>>1
        if arr[mid] < arr[mid+1] {
            l = mid+1
        } else {
            r = mid
        }
    }
    return l
}
```

写得比较糙可能不利于理解，主要是要睡觉了，有时间再来更新下。