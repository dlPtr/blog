---
date: 2022-02-13T18:24:59+08:00
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
title: "用 rand7() 实现 rand10() "
description: ""
tags: ["algo"]
categories: ["posts"]
draft: false
cover:
    image: "http://120.76.102.194/pub/imgs/rand7ToRand10.jpg" # image path/url
    # alt: "should have a png here" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page

---

### 题目
给定方法`rand7` 可生成`[1,7]`范围内的均匀随机整数，试写一个方法 `rand10` 生成 `[1,10]` 范围内的均匀随机整数。

你只能调用 `rand7()` 且不能调用其他方法。请不要使用系统的 `Math.random()` 方法。

每个测试用例将有一个内部参数 `n`，即你实现的函数 `rand10()` 在测试时将被调用的次数。请注意，这不是传递给 `rand10()` 的参数。

> 来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/implement-rand10-using-rand7
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。


### 题解
 
原理：`(randX() - 1)*Y + randY()`可以等概率的生成`[1, X * Y]`范围的随机数

rand7()可以生成随机数范围为：`[1,7]`，那么我们将其乘以7之后，映射为：`{7,14,21,28,35,42,49}`，每个数字出现的概率均为`1/7`，且相邻数字之间的差为`7`，那么再加上一个`rand7()`，即可填充缝隙，且根据独立概率公式：
> P(AB)=P(A)*P(B)

可证，填充缝隙后，`[8,56]`这`49`个数出现的概率均相同，为`1/49`。

由于这个区间不太好转换为`[1,10]`，所以在一开始对`rand7()`乘7前，就将其减去1，如下：
> (rand7()-1)*7 + rand7()

得到的区间为：`[1,49]`，舍弃`[41-49]`，随即得到的数模`10`，然后`+1`，区间即为：`[1,10]`。

```go
func rand10() int {
	for {
		ans := (rand7()-1)*7 + rand7()
		if ans	<= 40 {
			return ans%10 + 1
		}
	}
}
```

### 扩展：优化思路

由于产生的区间为：`[1,49]`，且我们舍弃了`[41,49]`这个区间，跑到这个区间里的概率是：`9/49`，还是较大的，那么我们对这个区间再做一次映射，
> `大于40的随机数-40-1`*7+ rand7()

如此就将其映射到了区间：`[1,63]`，那么这个时候被舍弃的就是：`[61,63]`这个区间，舍弃的是3个数，当然还可以再来一轮，
> `大于60的随机数-61`*7+rand7()

得到区间：`[1,21]`。
如果运气太差，恰巧得到的就是21，此时仅有一个数，已经没有了任何用处也无法再往下映射。
那么就重开吧，重新生成`[1,49]`的随机数。

```go
func rand10() int {
    for {
        ans := (rand7()-1) * 7 + rand7()
        // ans: [1,40]
        if ans <= 40 {
            return ans%10+1
        }

        // ans: [41,49]
        ans = (ans-41)*7 + rand7() // now ans: [0,63]
        if ans <= 60 {
            return ans%10+1
        }

        // ans：[61-63]
        ans = (ans-61)*7 + rand7() // ans: [1,21]
        if ans <= 20 {
            return ans%10+1    
        }
        
        // ans:[21]
        // 拒绝采样，重新执行一次循环
    }
}
```