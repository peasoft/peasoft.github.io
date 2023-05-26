---
layout: post
title: '技术手段破解手游防沉迷限制'
date: 2023-01-06
categories: 技术
tags: Android 破解 逆向 吾爱破解论坛
---

此文转自吾爱破解论坛，作者就是我本人，[点击这里查看原文](https://www.52pojie.cn/thread-1733855-1-1.html)

# 0. 前言

前几天有坛友成功破解了页游的防沉迷限制，既然页游已经被破解成功了，那我们是不是也可以逝逝看破解手游呢？

# 1. 分析

先在手机上下载好游戏，使用MT管理器**提取出 APK，传到电脑上**。为了反编译和阅读代码，我们**在电脑上打开 JADX，把传到电脑上的游戏 APK 文件拖入窗口**。

![代码文件结构](https://cdn.staticaly.com/gh/peasoft/picx-images-hosting@master/20230526/code_tree.5jmpgp86mb80.webp)

从游戏的代码结构中可以看出：这个程序使用了某 SDK 来实现防沉迷检测。一般来说，**SDK 中有一个函数来判断能否进行游戏，游戏再根据这个函数的返回值选择进入或退出**。因此，第一种思路就是：**修改判定函数的返回值**。那么，我们怎样找到判定函数呢？我们可以参考 SDK 的开发文档：aHR0cHM6Ly9kZXZlbG9wZXIudGFwdGFwLmNvbS9kb2NzL3Nkay9hbnRpLWFkZGljdGlvbi9ndWlkZS8jJUU1JTlCJTlFJUU4JUIwJTgzJUU3JUIxJUJCJUU1JTlFJThC

在文档中，我们看到这个函数有多种回调状态码。因此，如果一个函数中同时包含了这几种状态码，这个函数应该就是判定函数。**我们在 JADX 中选择”导航“\-\>”搜索文本“，填入 `9002`。**

思考：为什么选择搜索`9002`？我们设想一下，假如判定函数可能给出的状态码有`1`和`9002`，搜索哪一个数有更大概率找到这个函数？应该选`9002`。`1`可以表示第1个变量(`v1`)、第1个寄存器(`p1`)等，在代码中**太常见**了，没有针对性。但是一个正常的函数不会含有`9002`个变量，这时`9002`肯定表示一个**特殊**的状态码。

![代码搜索结果](https://cdn.staticaly.com/gh/peasoft/picx-images-hosting@master/20230526/code_search.18tckozo8g2o.webp)

我们可以找到好几条可疑代码，我已经在上图中框出来了。我们重点分析框中的第2条。程序的流程大致是这样的：**先判断实名认证是否成功**(`状态码==500`)，如果不成功则弹出错误信息；如果成功则**判断用户是不是成年人**(`currentUserAgeLimit()==18`)，是就开始游戏，否则执行其它验证(?)。我们可以在检查状态码前把状态码强行赋值成500，或者在进入函数后直接跳转到“开始游戏”。但可惜的是，这两种破解方法都失败了。

# 2. 再次分析

我们在手机上用MT管理器打开 APK 中的 `classes.dex` 文件，切换到“常量”页面，选择“过滤”，输入“`https://`”。翻看结果，有没有发现“`https://tds-tapsdk.cn.tapapis.com`”？这个网址在上面提到的 SDK 文档的“REST API”部分也出现了。根据文档的说法，这个地址是用来检查玩家当前能否游戏的。那么，**如果我们把这个地址改掉，让这些请求全部发到我们自己的电脑上，再返回伪造的“验证通过”消息，不就能骗过游戏了**？我们点击这条文本，**把编辑框内的文本替换成“`http://电脑IP:8080`”**。确定，再应用修改，最后重编译 dex 及 APK，然后安装这个APK。（注意：如果手机没有Root并安装核心破解，需要先卸载原应用再安装。另外，因为游戏被修改过了，所以系统可能会报毒，无视即可）

接下来，我们要在电脑上启动一个服务器来接收游戏的实名认证请求，本文使用的是Python3+Flask。先创建文件 `server.py`，在文件中加入以下代码：

```python
from flask import Flask, request
app = Flask(__name__)
```

~~然后再创建 `go.bat`，用于快速启动 Flask 服务端：~~

```batch
@set FLASK_APP=server.py
@set FLASK_ENV=development
@py -3 -m flask run --host=0.0.0.0 --port=8080
```

~~运行 `go.bat`~~

**2023/05/26更新**：不用创建 `go.bat`，只要在 `server.py` 的代码结尾加上：

```python
if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

即可。

运行 `server.py`，观察控制台日志，游戏请求了以下地址：`/anti-addiction/v1/clients/<clientId>/configuration`（其中`<XXXXXX>`表示信息经过脱敏处理），同时游戏报错“无法连接网络”。我们应该返回什么信息给游戏呢？**我们把这个地址拼到原来的 API 域名后面，粘贴到浏览器打开**。我们得到了一大串内容，其中大多数信息已被我省略：

```json
{"success":true,"data":{"child_protected_config":{"share_time_switch":1,"use_time_switch":1,"no_identify_time":3600,"charge_amount_switch":1,"child_common_time":0,"child_holiday_time":3600,"night_strict_start":"21:00","night_strict_end":"20:00","night_strict_warn":1200,"remain_time_warn":2400,"upload_all_data":"0"},"name":<游戏名>,"upload_user_action":"1","ui_config":{"pay_limit_words":<"消费金额已达上限"提示文本>,"health_reminder_words":<"健康游戏"提示文本>,"auth_identify_words":<"实名认证"界面文本>},"register_config":{},"holiday":[<今年所有休息日的日期>]}}
```

然后我们给服务器加3行代码（**2023/05/26更新**：以下所有代码均加在 `if __name__ == '__main__'` 前）：

```python
@app.route('/anti-addiction/v1/clients/<clientId>/configuration')
def configuration(cliendId):
    return r'<把你得到的完整信息粘贴到这里>'
```

在游戏中选择“重试”。接着，游戏又请求了 `/real-name/v1/clients/<clientId>/users/<userIdentifier>`。我们再按照刚才的方法添加3行代码（这次得到的信息很短，我就直接把信息和代码一起给了）：

```python
@app.route('/real-name/v1/clients/<clientId>/users/<userIdentifier>')
def real_name(clientId, userIdentifier):
    return r'{"success":true,"data":{"status":2,"anti_addiction_token":"","age_limit":-1,"has_auth_record":false}}'
```

接着，游戏弹出了实名认证提示。我们随便填写姓名，身份证号也可以随便填，但要符合身份证号的格式。比如，你可以填 `110101190001011017`（免责声明：此处的身份证号是随机生成的，仅用作测试用途，如此号码真实存在纯属巧合！）我们的服务器接收到了一个 POST 请求（API 地址见代码）。接着再加几行代码：

```python
@app.route('/real-name/v1/clients/<clientId>/users/<userIdentifier>/manual', methods=["POST"])
def real_name_post(clientId, userIdentifier):
    print(request.data)
    return r'{}'
```

再次提交实名信息，控制台会输出游戏的请求信息，再把这些信息 POST 给真服务器，我们又得到了和刚才相同的数据。我们把数据填到添加的 `return` 语句中，在游戏中再次提交信息。这次，游戏提示“请输入真实信息”。那么，究竟要返回怎样的信息才能使游戏认为实名认证通过了呢？我们再仔细观察一下这串信息：

```json
{"success":true,"data":{"status":2,"anti_addiction_token":"","age_limit":-1,"has_auth_record":false}}
```

很显然，最前面的 `success` 是正常的，而 `data` 中的 `age_limit` 引起了我们的注意。还记得吗，再第一次分析中我们提到游戏会判断用户是不是成年人，即 `currentUserAgeLimit()` 是否为`18`。我们凭直觉判断，这个函数的返回值应该就是 `age_limit` 的值。所以“正确”的数据中 `age_limit` 应该就是`18`。而这里的 `status` 也需要调整。经测试，`status>=2`代表提交的是假信息；`1`代表信息正在审核；`0`代表实名认证已通过。另外2个数据不修改也没有问题。所以，最终的服务器代码应该是这样：

```python
@app.route('/real-name/v1/clients/<clientId>/users/<userIdentifier>/manual', methods=["POST"])
def real_name_post(clientId, userIdentifier):
    return r'{"success":true,"data":{"status":0,"anti_addiction_token":"","age_limit":18,"has_auth_record":false}}'
```

这时，游戏发出了最后一个 GET 请求：`/anti-addiction/v1/clients/<clientId>/users/<userIdentifier>/playable`。而响应这个请求非常简单，因为开发文档已经给出过样例了！（开发小哥：我给你样例是让你干这事的？~~原地去世~~）代码：

```python
@app.route('/anti-addiction/v1/clients/<clientId>/users/<userIdentifier>/playable')
def playable(clientId, userIdentifier):
    return r'{"success":true,"data":{"code":200,"can_play":true,"message":"游戏时间不受限制","remain_time":60,"cost_time":0,"restrict_type":0,"title":"健康游戏提示","description":"当前为成年人账号"}}'
```

最后，再次提交信息，**验证通过！成功进入游戏！！！~~全体起立！~~至此，防沉迷限制破解成功！！！**

不过，还可以优化！

# 3. 优化

如果你对着控制台继续观察一段时间就会发现，游戏还会向服务器请求 `server-time`，不过404好像没有影响游戏正常运行，所以我们不管了。游戏有时会用 POST 方式发出 `playable` 请求，所以我们可以让服务器也响应 POST。另外，上文中说到 `real_name` 请求和 `real_name_post` 请求时，真服务器返回的数据是一样的，因此我们不妨猜想一下，假如 `real_name` 通过了，游戏应该就不会再发送 `real_name_post` 请求了，所以我们可以把两个处理函数合二为一。最终代码如下：

```python
from flask import Flask

app = Flask(__name__)

@app.route('/anti-addiction/v1/clients/<clientId>/configuration')
def configuration(clientId):
    return r'<把你得到的完整信息粘贴到这里>'

@app.route('/real-name/v1/clients/<clientId>/users/<userIdentifier>')
def real_name(clientId, userIdentifier):
    return r'{"success":true,"data":{"status":0,"anti_addiction_token":"","age_limit":18,"has_auth_record":false}}'

@app.route('/anti-addiction/v1/clients/<clientId>/users/<userIdentifier>/playable', methods=['get','post'])
def playable(clientId, userIdentifier):
    return r'{"success":true,"data":{"code":200,"can_play":true,"message":"游戏时间不受限制","remain_time":60,"cost_time":0,"restrict_type":0,"title":"健康游戏提示","description":"当前为成年人账号"}}'


# 2023/05/26更新：不用创建 `go.bat`，加入以下代码即可
if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

最终的控制台日志截图：

![控制台日志](https://cdn.staticaly.com/gh/peasoft/picx-images-hosting@master/20230526/server.e5rj3svz99c.webp)

# 4. 总结

看完这篇文章，你或许已经能够破解使用同样的 SDK 制作的其它游戏了，因为本文针对的其实并不只是单独一款游戏，而是这个 SDK。但我并不希望一些熊孩子学会破解。顺便说一下，这种通过**把 API 调用转移到自己的服务器上并返回假信息**的方法被我自己称为模拟验证破解法，而这种思路还能用于破解登录限制，破解内购等一系列其它限制。欢迎大家分享自己的其它思路！