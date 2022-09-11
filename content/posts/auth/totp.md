---
date: 2022-09-04T17:48:58+08:00
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
title: "TOTP 认证原理"
description: ""
tags: ["auth"]
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

### 一、背景：双因素认证

认证是大部分应用程序必不可少的一个关键环节，网站、App 等基于认证结果，确认用户的身份并为之分配相应权限。
密码认证是最常见的认证方式，但是不安全，容易泄露或被窃取。

![](/static/images/2022-09-11-15-02-34.png)

常用认证因素有以下三种：
- 秘密信息（所知）：只有用户知道，其它人不知道的信息，例如密码。
- 个人物品（所持）：用户持有的私人物品，例如 手机卡、身份证、USB key。
- 胜利特征（所有）：用户的遗传特征，例如 指纹、虹膜、声纹

双因素认证，就是结合两种因素的认证，常见的比如现在许多网站，输入密码之后可能还会要求发送短信验证。

> 大家日常使用时有没有发现，微信认证也是双因素的，当使用新手机登录微信时，除了需要输入密码外，还需要通过验证来为新手机授信，证明手机是安全的。
> 通过验证后，后续使用同一手机登录时就不需要再重复验证了，相当于是一个基于`所持`的因素认证。

`OTP` 作为一种公认的可靠解决方案，成为了许多公司在实现双因素认证时优先考虑的一种方式，已经被写入国际标准：[RFC4226](https://www.rfc-editor.org/rfc/rfc4226) 。

`TOTP` 是基于 `OTP` 的一种实现，以 `TOTP` 举例，使用步骤如下：

1. 第一步，管理员在认证服务器生成一个密钥。
2. 第二步，用户首次认证（例如密码认证）通过后，服务器提示用户扫描二维码（或其他方式），将密钥保存到手机应用程序上。如果用户之前已经保存过密钥，则第二步会跳过。

![](/static/images/2022-09-11-16-35-53.png)

3. 第三步，用户打开应用程序获取密码。

![](/static/images/2022-09-11-15-42-52.png)

4. 第四步，用户输入应用程序上生成的密码点击提交，认证服务器也会根据当前时间以及第一步生成的密钥，生成一个密码和用户输入的密码进行比对，相同则认证通过。

![](/static/images/2022-09-11-15-44-19.png)

### 二、OTP (One-Time Password) 原理

#### 2.1 变量定义：

|变量名|含义|
|-|-|
|`HMAC`|HMAC函数，一般使用 HMAC-SHA1|
|`K`|密钥|
|`C`|计算因子|
|`Truncate`|工具函数，哈希串太长不利于生成固定长度的密码，需要进行截断，截断规则由 [RFC4226](https://www.rfc-editor.org/rfc/rfc4226) 定义|

#### 2.2 算法步骤：

> OTP(K, C) = Truncate(HMAC-SHA-1(K, C))

我们可以用三个步骤来描述密码生成的过程：

**第一步**：基于 `HMAC` 算法生成哈希值。

在 [RFC4226](https://www.rfc-editor.org/rfc/rfc4226) 文档中官方使用的算法是 `HMAC-SHA1` ，基于 `K` 对 `C` 做哈希，生成的摘要长度有 `160` 比特，即 `20` 字节。

> HS = HMAC-SHA1(K, C)

**第二步**：截取 `HS` ，得到一个 `4` 字节长度的串。

由于 `HS` 的长度为 `20字节` ，对于用户来说，输入这么长一个密码不太方便还容易输错。

于是，在这一步 `Truncate` 函数会取 `HS` 的最后一个字节的低 `4` 位，作为偏移量（`0-15`）。根据偏移量从 `HS` 中取出四个字节，按照大端方式组成一个4字节的整数。

为确保最后得到的是一个正数，最高位会固定为0。

```golang
func Truncate(hmacHash) string {
    offset := int(hmacHash[len(hmacHash)-1] & 0xf)
    code := ((int(hmacHash[offset]) & 0x7f) << 24) |
        ((int(hmacHash[offset+1] & 0xff)) << 16) |
        ((int(hmacHash[offset+2] & 0xff)) << 8) |
        (int(hmacHash[offset+3]) & 0xff)

    ...step3...
}

```

**第三步**：第二步结束后，我们已经得到了一个 `4` 字节的正整数。然而这样还是不便于用户输入，所以还要做一次转换。

```golang
func Truncate(hmacHash) string {
    ...step2...

    // 做模运算，舍去超过 nDigits 位数的数字
    code = code % int(math.Pow10(nDigits))
    // 转换成字符串，加前缀0以确保长度为 nDigits
	return fmt.Sprintf(fmt.Sprintf("%%0%dd", nDigits), code)
}
```

### 三、TOTP 原理

看完了 `OTP` 算法流程，其实 `TOTP` 差不多也就是这样。

`TOTP` - `Time-Based One-Time Password`，即 基于时间的一次性密码。

其计算因子 `C1` 是一个时间单位，它将时间戳分为若干个区间，区间的步长一般为 `30s`。

> C1 = UNIXTimeStamp / 30

加入步长是因为从 用户获取密码 到 输入、提交、服务器完成校验 也是需要一定时间的。如果直接用时间戳的话，那么必须在 `1s` 内完成这些操作。加入步长后，只要用户是在 `C1` 区间内完成即可。

```txt
0s               30s    30xC1 s          30x(C1+1) s
|       30s      |......|       30s      |       30s        |
|------[C1-1]----|......|------[C1]------|------[C1+1]------|
```

### 四、otp密钥传递

第一节中，我们提到服务器将密钥传递给用户时，是以二维码形式提供的。

![](/static/images/2022-09-11-16-40-16.png)

那么二维码中都包含了哪些信息呢？

扫码得到内容如下:

> otpauth://totp/ACME:john@example.com?secret=X54T3TV7K7MYLKV5HJQLQIUVGZEJRHT3&issuer=ACME&algorithm=SHA1&digits=6&period=30

我们定义如下规则:

> otpauth://TYPE/LABEL?PARAMETERS

划分各部分后得到：

<table>
    <tr>
        <td colspan=2> 字段 </td>
        <td colspan=2> 取值 </td>
        <td> 含义 </td>
    </tr>
    <tr >
        <td colspan=2> TYPE </td>
        <td colspan=2> totp </td>
        <td> otp类型，取值范围：totp/hotp </td>
    </tr>
    <tr>
        <td colspan=2> LABEL </td>
        <td colspan=2> ACME:john@example.com </td>
        <td> 密钥由谁(ACME)颁发给谁(john@example.com) </td>
    </tr>
    <tr>
        <td rowspan=6 colspan=2> PARAMETERS </td>
        <td> key </td>
        <td> value </td>
        <td>-</td>
    </tr>
    <tr>
        <td> secret </td>
        <td> X54T3TV7K7MYLKV5HJQLQIUVGZEJRHT3 </td>
        <td> （必填）密钥 </td>
    </tr>
    <tr>
        <td> issuer </td>
        <td> ACME </td>
        <td> （选填，但建议填写）颁发者，如果此字段未定义，则尝试从 LABEL 中解析 </td>
    </tr>
    <tr>
        <td> algorithm </td>
        <td> SHA1 </td>
        <td> （选填）哈希算法</br> <li>SHA-1（默认）</li><li>SHA-256</li><li>SHA-512</li> </td>
    </tr>
    <tr>
        <td> digits </td>
        <td> 6 </td>
        <td> （选填）密码位数，默认为 6 </td>
    </tr>
    <tr>
        <td> period </td>
        <td> 30 </td>
        <td> （选填-仅适用于 totp）时间步长，默认 30s </td>
    </tr>
    
</table>

详细规则见：[https://github.com/google/google-authenticator/wiki/Key-Uri-Format](https://github.com/google/google-authenticator/wiki/Key-Uri-Format)

### 五、安全性分析

TODO

### 六、扩展：HOTP 认证

TODO
### 七、总结

TODO

### 八、参考资料

TODO