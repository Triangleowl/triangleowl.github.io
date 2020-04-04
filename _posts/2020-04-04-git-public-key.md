---
layout: post
title: 生成 SSH 公钥
author: Triangleowl
categories: git
permalink: /:categories/:title
---

生成 `git` 公钥如下：

```shell
$ ssh-keygen
```

执行结果：
```shell
$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/sl/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/sl/.ssh/id_rsa.
Your public key has been saved in /home/sl/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:XXXrY7BeVPe6Sx3ZPXXXI/7pZS9DWT7SOXXXeO/TY+M sl@thinkpad
The key's randomart image is:
+---[RSA 2048]----+
|   .... .        |
|  ..o  . .       |
|   +..    . . o  |
|  . *. . . + + o.|
| . + .. S o o +++|
|  .      o . oo@.|
|        o . ..O B|
|       . . . .o%o|
|        .   .o*E=|
+----[SHA256]-----+

```

然后进入 ~/.ssh 目录，将
```shell
$ ls
config  id_rsa  id_rsa.pub  known_hosts

$ cat id_rsa.pub
    ......
```

将id_rsa.pub内的内容全部拷贝到github上，点击添加即可。