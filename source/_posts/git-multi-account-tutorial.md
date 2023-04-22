---
title: git-multi-account-tutorial
comments: false
aplayer: false
tags:
  - git
abbrlink: '3e482544'
date: 2023-04-23 12:01:52
categories:
keywords:
description:
top_img:
cover:
sticky:
---
#### 1、如果设置过全局的`username` 和`email`，清空全局配置的`username`和`email`

+ 查看git全局config配置
```shell
git config --list
```
+ 清空默认邮箱和用户名
```shell
git config --global --unset user.name
git config --global --unset user.email
```
#### 2、为不同的git账户生成RSA公私钥
+ 生成密钥
```shell
ssh-keygent -t rsa -C "[email-address]"
```
eg:
```shell
ssh-keygen -t rsa -C "huan.wenlong@gmail.com"
```
输出如下：
```shell
Generating public/private rsa key pair.
Enter file in which to save the key (/c/Users/huan.wenlong/.ssh/id_rsa):
```
+ 输入密钥名称
**此时输入文件名称，多个账户时名称不能重复**

+ 接下来配置密码，如果需要设置可以输入密码，如果不需要一路回车即可
---

以下以配置`GitHub`和`gitee`为例
+ GitHub
```shell
ssh-keygen -t rsa -C "huan.wenlong.529@gmail.com"
Generating public/private rsa key pair.
Enter file in which to save the key (/c/Users/huan.wenlong/.ssh/id_rsa): github
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in github
Your public key has been saved in github.pub
The key fingerprint is:
SHA256:zlgu9ku5ptduRu3gf/ZJDKCf7E3XiVkBZwLiSR2WaV8 huan.wenlong.529@gmail.com
The key's randomart image is:
+---[RSA 3072]----+
|         o.+=o o |
|        o +=  =E |
|         o... .. |
|          . ..  .|
|        S. . . . |
|       * .= o * o|
|      + *+ * + =.|
|     . +o.* + = .|
|      .++=.o.+ o.|
+----[SHA256]-----+
```

+ gitee
```shell
$ ssh-keygen -t rsa -C "381897190@qq.com"
Generating public/private rsa key pair.
Enter file in which to save the key (/c/Users/huan.wenlong/.ssh/id_rsa): gitee
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in gitee
Your public key has been saved in gitee.pub
The key fingerprint is:
SHA256:sCDg0WghkvmprbjmzFm0PkzseLss+O8z2PtQgAtoqjw 381897190@qq.com
The key's randomart image is:
+---[RSA 3072]----+
|+=+              |
|B+.o             |
|++o.o .          |
|o.oo o o         |
|.oo.  o S        |
|+ oo..           |
|+E==.            |
|*+B=+.           |
|+B+BB=.          |
+----[SHA256]-----+
```

#### 3、配置`GitHub`或`Gitee`公钥
![github](/resources/assets/3e482544/20230423152155.png)
![gitee](/resources/assets/3e482544/20230423152343.png)

将刚才生成的密钥以文本方式打开复制到`github`或`gitee`中添加密钥即可。

#### 4、配置git config
+ 打开`~/.ssh`目录
+ 查看是否由`config`文件，有则修改，没有则添加
以下为示例，具体可参考 [sshconfig](https://man7.org/linux/man-pages/man5/ssh_config.5.html?spm=a2c4g.322237.0.0.6760114agqwpwm)
```text
Host 别名
HostName 域名
User 用户
IdentityFile 密钥路径
PreferredAuthentications 认证首选
```
以下为添加`GitHub`和`gitee`的示例
```text
Host github
HostName github.com
User git
IdentityFile ~/.ssh/github
PreferredAuthentications publickey

Host gitee
HostName gitee.com
User git
IdentityFile ~/.ssh/gitee
PreferredAuthentications publickey
```

#### 5、添加RSA公钥到信任列表
+ 确保`ssh-agent`处于运行中，如果未运行需要运行`ssh-agent`
Windows下运行`ssh-agent`如下
打开服务，找到`OpenSSH Authentication Agent`设置为自动运行
![ssh-agent](/resources/assets/3e482544/20230423154352.png)

+ 添加RSA到SSH agent

```shell
ssh-add ~/.ssh/[name]
```
其中`name`为刚才生成的文件名称
此处以添加`GitHub`和`gitee`为例
```shell
ssh-add ~/.ssh/github
ssh-add ~/.ssh/gitee
```

#### 6、测试连接
+ 单账号
```shell
ssh -T git@gitee.com
```

+ 多账号
```shell
ssh -T git@[config 中的user]@[config 中的host]
```

eg：gitee
```shell
ssh -T git@git@gitee
```
第一次会出现不能建立连接，询问是否继续，输入`yes`回车即可
```shell
The authenticity of host 'gitee.com (212.64.63.190)' can't be established.
ECDSA key fingerprint is SHA256:FQGC9Kn/eye1W8icdBgrQp+KkGYoFgbVr17bmjey0Wc.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'gitee.com,212.64.63.190' (ECDSA) to the list of known hosts.
```

看到如下信息就说明成功了
```shell
Hi Monster! You've successfully authenticated, but GITEE.COM does not provide shell access.
```

----
不知道github为啥不能带用户，后面有时间再研究下
```shell
$ ssh -T git@git@github
git@git@github.com: Permission denied (publickey).
```
去掉用户名验证成功

```shell
$ ssh -T git@github
Hi wen-loong! You've successfully authenticated, but GitHub does not provide shell access.
```