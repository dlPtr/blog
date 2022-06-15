---
date: 2022-06-15T23:58:48+08:00
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
title: "排障：Egg-Sequelize连接mysql失败 - (using password: NO)"
description: ""
tags: ["troubleshooting"]
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


## 1. 背景

学习 `egg-js` 时，使用 `egg-sequelize` 连接 `mysql` 。

数据库 `migration` 成功，`user` 表成功建立，证明实际上是可以成功和 `mysql` 建立连接的。
在 `npm run dev` 启动项目时，却始终报错：

![x](/images/2022-06-16-02-12-14.png)

按照官方文档(https://www.eggjs.org/zh-CN/tutorials/sequelize)，如下：

![x](/images/2022-06-16-02-15-45.png)

我已经在 `database/config.json` 中配置了 `mysql` 的用户名和密码，按理说是不会有如下报错的：

> WARN 671980 Sequelize Error: Access denied for user 'root'@'localhost' (using password: NO)

## 2. 解决过程

### 2.1 首先想到的是先谷歌搜一下报错信息

搜到的一堆都是让开 `root` 远程登录权限的，登录带参数 `-p` 之类的。似乎没有人遇到类似的 `egg-sequelize` 登录问题。


### 2.2 排除法，定位问题是在服务端还是客户端

尝试使用 `mysql-workbench` 指定账号为 `root`，输入密码尝试连接 `mysql`, 成功。
说明问题应该不在于服务端，远程登录是OK的。

### 2.3 抓包对比分析

既然 `mysql-workbench` 可以连接成功，`egg-sequelize` 不行，那么抓包比对下，数据包的差异在哪。

#### 2.3.1 关掉ssl

```
1. 修改 /etc/my.cnf，加入:
[mysqld]
skip_ssl

2. 重启：
systemctl restart mysqld

3. 验证下是否关闭成功：
mysql> show variables like 'have_ssl%';
+---------------+----------+
| Variable_name | Value    |
+---------------+----------+
| have_ssl      | DISABLED |
+---------------+----------+
1 row in set (0.00 sec)

```


#### 2.3.2 mysql-workbench

![x](/images/2022-06-16-02-11-08.png)

#### 2.3.3 egg-sequelize

![x](/images/2022-06-16-02-11-36.png)

#### 2.3.4 猜测

确实没有传 `密码` ，那么猜了一下是不是配置错了？

再次打开 `eggjs` 官方文档看了下，发现有两处提到需要配置 `mysql` 连接参数。

第一处是：`database/config.json`，这个没问题，我们按照教程去改了：

也确实生效了，数据库初始化执行成功。
```
╭─root@localhost ~/workspace/code/bookRegister ‹master●› 
╰─# npx sequelize db:migrate

Sequelize CLI [Node: 16.15.1, CLI: 6.4.1, ORM: 6.20.1]

Loaded configuration file "database/config.json".
Using environment "development".
== 20220608163151-init-users: migrating =======
== 20220608163151-init-users: migrated (0.018s)

== 20220615163432-init-users: migrating =======
== 20220615163432-init-users: migrated (0.010s)

== 20220615163524-init-users: migrating =======
== 20220615163524-init-users: migrated (0.009s)
```

从日志可以看出，使用的是 `database/config.json` 的配置。

而第二处，在另外一个地方：`config/config.default.js`，官方给的示例中未填写 `用户名` 和 `密码`，且也未做解释。

![](/images/2022-06-16-02-35-09.png)

尝试在这份配置里也加上 `username` 和 `password`，果然在项目运行时，使用的是这份配置中的参数。

![](/images/2022-06-16-02-30-46.png)

实在是神坑，首先为啥两个地方都要配置，不能复用吗？
其次，或许是因为一些框架原因，导致数据库 `migration` 和 `运行时` 用的配置不一致，但起码也得在文档中告知一下呀。。。