---
date: 2021-12-08T16:11:24+08:00
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
ShowBreadCrumbs: true
ShowPostNavLinks: true
author: "robinx"

# TODO
# weight: 3 # 优先级
title: "Yoda-Condition：让代码可读性变差的一种表达式写法属于是"
tags: ["code-style", "misc"]
categories: ["misc"]
draft: false
cover:
    # image: "/postImages/yoda-speak.png" # image path/url
    # alt: "yoda speak" # alt text
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page

---

## c语言中的yoda-condition写法

大学期间学习c语言时发现过一个问题。

c语言允许在判断条件中进行赋值，并且以赋值后的变量作为判断条件。
这就导致，c语言初学者(熟练者也偶尔会犯错误)常因为少写一个`=`号引发逻辑bug，让人怀疑人生。如下：
```c
int age = 24;

// 正确的写法
if (age == 23) {
    // age == 23为假
}

// 误写为
if (age = 23) {
    // age被赋值为23，导致条件为真
}

```
错误的分支中，age被赋值为23，非0值作为判断条件为真，故条件成立，进入if分支。

当时在网上搜索过这个问题，发现有一种写法可以在编译阶段就暴露这个问题，就是将常量放在 = 左边，变量放在右边。这样如果少写了一个 = ，就从关系运算符变成了赋值操作符，静态分析过程中就可以发现此问题。
```c
if (23 = age) {
    // 编译阶段会报错
}
```


## yoda-condition写法应该被摈弃
今天在用golang写一个demo时，不小心把常量放到了 != 左边，导致go-staticcheck报错：

![hhhh](/images/yoda-speak.png)

搜了一下，如今这种常量在左的写法已经不再受推崇了，这等于是牺牲了一部分代码可读性来换取代码逻辑的正确，现在有各种牛逼的插件可以做静态分析，所以感觉已经没必要了。

而且其实当初我模仿这种写法的时候，确实觉得很别扭，因为我想表达的是`年龄等于23`，而不是`23等于年龄`，这和我们阅读习惯不一致。

## 为什么叫yoda-condition？

搜了下才知道，yoda是星球大战中的一个角色-尤达大师。喜欢说倒装句，我们把关系运算符左右的变量和常量互换，就像是用编程语言中的倒装一样。


搜了一下，如今这种常量在左的写法已经不再受推崇了，这等于是牺牲了一部分代码可读性来换取代码逻辑的正确，现在有各种牛逼的插件可以做静态分析，所以感觉已经没必要了。

而且其实当初我模仿这种写法的时候，确实觉得很别扭，因为我想表达的是`年龄等于23`，而不是`23等于年龄`，这和我们阅读习惯不一致。
搜了一下，如今这种常量在左的写法已经不再受推崇了，这等于是牺牲了一部分代码可读性来换取代码逻辑的正确，现在有各种牛逼的插件可以做静态分析，所以感觉已经没必要了。
