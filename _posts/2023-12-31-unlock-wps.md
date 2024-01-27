---
layout: post
title: '最新破解 WPS 2023 个人版不登录不能用限制'
date: 2023-12-31
categories: 技术
tags: 破解 逆向 吾爱破解论坛
---

不知从哪一版开始，WPS 必须登录后才能使用了，这给公用电脑带来了很大的麻烦。那么，能不能解除这一限制呢？

当然可以！只要打开 WPS 配置工具，选择高级、其他选项、兼容异常网络环境……不登录（1 天（新版为 3 天）内有效）……

等等？只能跳过 3 天？没有 KMS38 我是不会罢休的！

我们要对 WPS 的注册表下手。首先，导出注册表：`HKEY_CURRENT_USER\Software\kingsoft\`。然后，打开 WPS 配置工具，解除限制。再次导出注册表。使用工具对比前后导出的两份注册表，结果如图：

![对比结果](https://testingcf.jsdelivr.net/gh/peasoft/picx-images-hosting@master/20240127/对比结果.ihmsz5rekeg.webp)

其中 `forced` 项与 `lastSkipVistorModeTime` 被修改了，从名称推断，这两项很可能与登录限制有关。另外还有一个项 `isForceLoginNewDevice202111`，这个项没有被修改，但看着也有点意思。下面我们分别对这三项进行修改。注意：每次测试另一项前要把上一项的修改还原！

1. 将 `forced` 项的值 `c2tpcFZpc3Rvck1vZGUuMTcwMzIwMzYzNQ==` 进行解码，得到 `skipVistorMode.1703203635`，其中 `1703203635` 是一个 Unix 时间戳或称 C 时间戳，转换为年月日时分秒后就是我们点击“跳过登录”的时间。将时间戳改为 9999 年后回编码，修改注册表，无效。（这里我没有确认是因为时间戳过大导致 WPS 无法识别还是这一项本来就无关紧要，欢迎热心坛友进一步测试！）
2. 将 `lastSkipVistorModeTime` 的值 `6584d333` 转换成十进制，得到 `1703203635`，再进行时间戳转换，发现这就是我们跳过登录的时间。WPS 会将当前时间与这个时间相减，如果差值在 1 天（或 3 天）之内，则“跳过登录”仍然有效。因此，我们把这项改成 `ffffffff`，即 `2106/02/07 14:28:15`，我们就可以跳过登录 82 年了。经测试，修改有效。
3. 由名称 `isForceLoginNewDevice202111` 推测，这一项指示了 WPS 是否是从 2021.11 版之前的版本升级而来的，如果是，则仍能像老版本一样无需登录即可使用。将这一项的值从 `true` 改为 `false`。经测试，修改有效。

由此，我们制作出了以下注册表补丁：（2024/01/02 更新：可能要保存成 `UTF-16 LE (BOM)` 格式，但是默认存成 `ANSI` 似乎也没问题？**最后两项是根据回帖新增的**，分别兼容旧版 WPS 和教育版 WPS，在此感谢 107 楼）

```reg
Windows Registry Editor Version 5.00

[HKEY_CURRENT_USER\Software\kingsoft\Office\6.0\plugins\officespace\flogin]
"isForceLoginNewDevice202111"="false"
"lastSkipVistorModeTime"=dword:ffffffff
"enableForceLoginForFirstInstallDevice"="false"
"enableForceLoginStyle_202111"="false"
```

<https://peasoft.github.io/dl/UnlockWPS.reg>

导入后即可永久破解 WPS 的登录限制。

小声：值得注意的是，第 3 项方法在网上流传已久，要修改的注册项名以前叫 `enableForceLoginForFirstInstallDevice`（据坛友回复，现在这个项仍然有效），但是谁也说不准 WPS 将来流氓起来会不会对所有用户“一视同仁”。而第 2 项方法网络上尚未出现过。总算，这问题总算在今年解决了……

[本文在吾爱破解论坛同步发布，点此查看](https://www.52pojie.cn/thread-1875176-1-1.html
)
