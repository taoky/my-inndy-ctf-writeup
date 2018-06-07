# Misc

## 2: corgi can fly

可以使用 `StegSolve` 找到图片中的隐藏信息。这张图片在 Red plane 0 处有一张二维码，扫码即可。

## 3: television

下载得到的 BMP 文件中包含杂乱无章的像素点。然而，这道题不是去解出像素点包含的内容，而是

```
strings television.bmp 
```

（当然，用文本编辑器打开，然后搜索 FLAG，也是可以的）

## 4: meow

直接用 `StegSolve` 看是看不出东西的。使用 `binwalk` 可以发现里面还藏了一些东西。

提取，发现一个压缩包，是加密的。但可以看到压缩包中除了 flag，还有一张 Pusheen 的照片。

可以用 `foremost` 解出原始 png 图像，它的大小和压缩包中照片大小是相同的。可以猜想，这就是压缩包中的图像。

所以我们可以来已知明文攻击。（详见 [CTF Wiki](https://ctf-wiki.github.io/ctf-wiki/misc/archive/zip/#_6)）

对 zip 的已知明文攻击需要使用压缩文件的压缩工具。使用 `zipinfo -v` 查看压缩包，发现是使用 UNIX 下的工具压缩的。

将原图象文件放在与待解密压缩文件相同的目录树下。使用 `zip -r meow.zip meow` 进行压缩。然后拖到 ARCHPR 下即可。只需要恢复加密密钥就行了，原本的加密口令我跑了一会，没跑出来。

## 5: where is flag

解压缩，发现一堆乱七八糟的东西。

从第一题知道 flag 格式为 `FLAG{This is flag's format}`，所以使用 `egrep` 找。

`egrep -o 'FLAG{[a-zA-Z0-9]+}' flag`

## 6: encoder

解压缩得到一个 tar 格式文件，再解压缩，得到 `encoder.py` 和 `flag.enc`。`encoder.py` 为将 flag 文本加密为 `flag.enc` 的程序。

可以看到，`encoder.py` 中有四种「加密」方式，每次随机选取一种，进行 30 到 50 次不等的加密操作。每个文件的第一个数字代表选取的「加密」方式。

所以解密的话四个函数只要反过来写就可以了。

```python
def un_rot13(s):
    return s.translate(string.maketrans(string.uppercase[13:] + string.uppercase[:13] +
        string.lowercase[13:] + string.lowercase[:13], 
        string.uppercase + string.lowercase))

def un_base64(s):
    return ''.join(s.decode('base64').split())

def un_hex(s):
    return s.decode('hex')

def un_upsidedown(s):
    return s.translate(string.maketrans(string.lowercase + string.uppercase,
        string.uppercase + string.lowercase))

E = (un_rot13, un_base64, un_hex, un_upsidedown)

enc_flag = open('flag.enc', 'r').read()

while True:
    if enc_flag[0].isdigit() == False:
        print enc_flag
        break
    c = int(enc_flag[0])
    enc_flag = E[c](enc_flag[1:])
```

## 7: slow

可以观察到，远端程序验证 flag 的速度很慢，猜测是时序攻击，每增加一个正确的字符就增加一秒。由于没有小写字母和空格，速度可以快一些。在 pwntools 的帮助下可以写出这个程序。

```python
from pwn import *
import time
import string
import math

context.log_level = "debug"

charset = "_" + string.uppercase + string.digits
flag = "FLAG{2_SLOW_I_M_GOING_TO_"
max_time = 26

cache=[0 for i in range(1000)]

while True:
    for offset, i in enumerate(charset):
        if cache[max_time] > offset:
            continue 
        p = remote("hackme.inndy.tw", 7708)
        p.recvuntil("flag? ")
        t1 = time.time()
        p.sendline(flag+i+"}")
        s = p.recv()
        t2 = time.time()
        print "tried " + flag + i + " with time " + str(t2 - t1)
        cache[max_time] = offset
        if math.floor(t2 - t1) > max_time:
            max_time += 1
            print "success try"
            flag += i
            break
        elif math.floor(t2 - t1) < max_time:
            print "rollback"
            flag = flag[:-1]
            max_time -= 1
            break
        if s[0] != 'B':
            print flag
            quit()
        p.close()
```

在网络环境不稳定的地方，可能要试好几次。跑这个脚本需要很长的时间。

## 8: pusheen.txt

解压缩的文本中可以发现实心 Pusheen 和空心 Pusheen，猜想它们分别代表二进制的 0 和 1。观察到每个 Pusheen 占 16 行，所以就可以写脚本了。

```python
s = open("pusheen.txt", "r")
binary_string = ""
for offset, i in enumerate(s):
    if offset % 16 != 1:
        continue
    if i == '   ▌▒▒▀▄▄▄▄▄▀▒▒▐▄▀▀▒██▒██▒▀▀▄\n':
        binary_string += "1"
    else:
        binary_string += "0"
        
print(hex(int(binary_string, 2)))
```

把结果拖到一个十六进制编辑器里面，就能看到 flag 了。

## 9: big

在多次解压缩之后得到的 big 文件非常非常大（17.18 GB）。尝试看这个文件前面一小部分，都是 `THISisNOTFLAG{}`。

那么，估计是要使用 `grep` 来寻找正确的 flag 了。

`grep` 支持使用 `-v` 参数反向查找。所以可以：

```
grep -v "THISisNOTFLAG{}" big
```

就行了。

**注意：GNU grep 比 BSD grep 快很多。所以使用 BSD 类系统（包括 macOS）自带的 grep 会非常非常慢。**

