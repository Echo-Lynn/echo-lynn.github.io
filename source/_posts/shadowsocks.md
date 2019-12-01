---
title: 在 Ubuntu 上使用 Shadowsocks 搭建个人 VPN 服务器
date: 2019-12-1 15:34:00
categories:
- Tools
- Shadowsocks
---

# 写在前面

拥有一台海外云服务器能做什么？

能搭建个人专属的 VPN 服务器

搭建 VPN 服务器能做什么？

能。。。。爱搭不搭

<!-- more -->

# 前期准备

- 一台装有Ubuntu系统且拥有海外公网 IP 的云服务器

# 安装 Shadowsocks 服务端

首先通过 SSH 连接你的云服务器，如何连接服务器不是本文的重点，这里不予以赘述

如果服务器的系统是新装的，建议先更新一下源

```bash
apt-get update & apt-get upgrade
```

更新完成后，安装 pip

```bash
apt-get install python-pip
```

利用 pip 安装 Shadowsocks 服务端程序

```bash
pip install shadowsocks
```



# 配置 Shadowsocks 服务器

按照上文安装 Shadowsocks 完成后，我们直接可以通过命令的方式启动 Shadowsocks 服务，但笔者推荐利用读取配置文件的方式来启动，所以就不介绍命令行启动的方式了

在登录用户的目录下创建配置文件：

```bash
cd ~
mkdir shadowsocks
vim shadowsocks/config.json
```

键入以下内容（别忘记把带有$的变量替换成你自己的信息）：

```json
{
    "server": "0.0.0.0",
    "server_port": "8388",
    "local_address": "127.0.0.0",
    "local_port": "1080",
    "port_password": {
        "8387": "$your_password"
    },
    "timeout": 300,
    "method": "aes-256-cfb",
    "fast_open": false
}
```



`server `直接 `0.0.0.0` 指向当前机器

`server_port` 服务端端口

`local_address` 是本地代理的地址

`local_port` 本地代理的端口

`port_password` 是创建客户端端口号；key （如 "8387"） 是端口号，value 是该端口号对应的连接密码，这里可以创建多个账号，供多个用户连接

`timeout` 超时时间

`method` 加密方式

`fast_open` 是否使用 TCP 连接



# 启动服务

完成配置文件的编写后，我们就来启动它

```bash
ssserver ‐c ~/shadowsocks/config.json ‐d start
```



# 验证服务是否成功开启

我们首先通过查看 shadowsocks 的日志文件来验证服务是否启动成功

```bash
cat /var/log/shadowsocks.log
```

如果一切顺利的话，你将只看到这样一句日志

```
INFO     starting server at xxx.xx.xxx.xxx:8387
```

这说明你的服务启动成功了

如果看到以下包含以下字样的报错日志：

```bash
undefined symbol: EVP_CIPHER_CTX_cleanup
```

这个问题是由于在 openssl 1.1.0 版本中，废弃了 EVP_CIPHER_CTX_cleanup 函数，如官网中所说：

> EVP_CIPHER_CTX was made opaque in OpenSSL 1.1.0. As a result, EVP_CIPHER_CTX_reset() appeared and EVP_CIPHER_CTX_cleanup() disappeared.  

> EVP_CIPHER_CTX_init() remains as an alias for EVP_CIPHER_CTX_reset(). 

解决问题的办法就是：

1. 用 vim 打开文件：vim /usr/local/lib/python2.7/dist-packages/shadowsocks/crypto/openssl.py (该路径请根据自己的系统情况自行修改，如果不知道该文件在哪里的话，可以使用find命令查找文件位置)
2. 搜索cleanup，共有两处
3. 将第一处（2.8.2版本在52行）libcrypto.EVP_CIPHER_CTX_cleanup.argtypes = (c_void_p,) 改为libcrypto.EVP_CIPHER_CTX_reset.argtypes = (c_void_p,)
4. 将第二处（2.8.2版本在111行）libcrypto.EVP_CIPHER_CTX_cleanup(self._ctx) 
   改为libcrypto.EVP_CIPHER_CTX_reset(self._ctx)
5. 保存并退出
6. 重新执行启动命令，然后查看日志文件验证是否启动成功

# 客户端连接

完成 Shadowsocks 的服务端部署后，尝试一下客户端连接

## 下载客户端程序 Outline

下载地址：[https://getoutline.org/zh-CN/home]

Outline 同时支持 Android、IOS、macOS、Windows、Chrome OS、Linux 六大终端平台，本文以 IOS 为例，App Store 可以直接搜索到 Outline 下载

然后我们转到 Shadowsocks 官网教程：[https://shadowsocks.org/en/config/quick-guide.html]

根据我们刚才的配置生成一个可以供 Outline 识别的链接

1. 找到 Try it yourself 一栏，在 Plain 一行输入 ss://aes-256-cfb:$your_password@xxx.xx.xxx.xxx:8387

   `ss://` 表示使用 Shadowsocks 协议

   `aes-256-cfb` 表示加密方式

   `$your_password` 8387 端口对应的密码

   `xxx.xx.xxx.xxx` 你服务器的公网 IP

   `8387` 服务端配置的端口

2. 拷贝 Encoded 下自动生成的链接到你的手机

打开 Outline - 添加服务器 - 输入刚才拷贝的链接 - 添加完成后点击连接

Outline Logo 变成绿色后表示连接成功，这个时候可以查看自己手机的 IP 是否变成你服务器的公网 IP 来做最后一次验证本次 VPN 是否搭建成功，查看方式有很多种，这里不再赘述

# 最后

在墙外放飞自我吧！！！

本文分享到此结束，如果过程遇到什么问题，欢迎在下面评论区给笔者留言，并留下您的邮箱，方便笔者联系