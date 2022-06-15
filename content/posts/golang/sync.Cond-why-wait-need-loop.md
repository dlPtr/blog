---
date: 2022-02-17T23:22:24+08:00
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
title: "为什么 Sync.Cond.Wait()建议在循环中调用?"
description: "why sync.Cond better wait in loop with condition"
tags: ["golang"]
categories: ["posts"]
draft: false
# weight: 1
cover:
    # image: "http://120.76.102.194/pub/imgs/<modify_here>.png" # image path/url
    # alt: "should have a png here" # alt text
    caption: "" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
---


### 1、前言

> 一句话总结：sync.Cond 条件变量用来协调想要访问共享资源的那些 goroutine，当共享资源的状态发生变化的时候，它可以用来通知被互斥锁阻塞的 goroutine

未使用过这个库的话，可以先了解下其使用方法以及场景：

> - [Go sync.Cond](https://geektutu.com/post/hpg-sync-cond.html)
> - [如何正确使用sync.Cond](https://stackoverflow.com/questions/36857167/how-to-correctly-use-sync-cond)

用大白话说，就是多个 `goroutine` 等待 一个 `goroutine` 完成任务后通知它们。

### 2、为何建议在循环中调用 `cond.Wait()`，循环条件为未达到执行条件。

**使用了一下，个人认为有以下两个原因：**
1. 当 `消费者` 等待的数据已经准备好，那么就不需要等生产者提供 `广播`/`单播` 信号，可直接进行读取。
2. 收到的 `广播`/`单播` 信号不一定是消费者需要的那个，如果被唤醒后发现读取条件未满足，则继续调用 `Cond.Wait()`。

**具体可参考代码：**

> 可以直接在 [playgroud](https://go.dev/play/p/3qaz7-F0Ut-) 运行代码。

```go
package main

import (
	"fmt"
	"sync"
	"testing"
	"time"
)
func TestCond2(t *testing.T) {
	wg := &sync.WaitGroup{}
	cond := sync.NewCond(&sync.Mutex{})

	msg := ""
	ready := false

	// 生产者
	wg.Add(1)
	go func() {
		// time.Sleep(2 * time.Second) // 注释二
		defer wg.Done()
		fmt.Println("writer started")

		cond.L.Lock()
		msg, ready = "hey there!", true
		cond.L.Unlock()
		cond.Broadcast()
	}()

	// 消费者
	wg.Add(1)
	go func() {
		// 去掉注释一的 time.Sleep，即明白for循环的作用之一：
		// 如果数据已经准备好了，那么读者无需调用 Cond.Wait() 即可去读数据。
		// time.Sleep(1 * time.Second) // 注释一
		defer wg.Done()
		fmt.Println("reader started")

		cond.L.Lock()
		for !ready {
			fmt.Println("msg not ready, start wait")
			cond.Wait()
			fmt.Println("reader activated")
		}
		fmt.Println("msg from wirter:", msg)
		cond.L.Unlock()
	}()

	// 去掉注释二(74行、106-107行)，可以看到for循环的另一作用：
	// 如果收到广播信号，for 循环的条件可以校验当前是否满足读的条件
	// 如果不满足，则重新 Wait()。
	// time.Sleep(time.Second) // 注释二
	// cond.Broadcast()        // 注释二
	wg.Wait()
}
```