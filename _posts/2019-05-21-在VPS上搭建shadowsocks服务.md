# 2019-05-21-在VPS上搭建shadowsocks服务

### 1.一台VPS

搬瓦工涨价了，着实用不起。同事推荐了[justhost](https://justhost.ru/hosting)，俄罗斯的一家服务商，不用搭梯子就能访问。看自己需求，我买的是最便宜的配置：单核CPU，512Mb的RAM，5G硬盘，一年120RMB左右，不过结算是用卢布，需要一张境外支付卡（visa或者万事达...）。付款成功以后会给注册时候的邮箱发送邮件，里面有ssh登录的IP，账号等详细信息。顺带说下ssh工具，xShell个人版不能长期使用了，得破解，不过我基本不用破解版软件，所以找了个MobaXterm免费版，功能很强。省事的话PuTTY也行。

### 2.安装shadowsock服务

在网上找了段脚本:

#### 2.1 一行脚本安装服务

```shell
wget --no-check-certificate -O shadowsocks-all.sh https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocks-all.sh
chmod +x shadowsocks-all.sh
./shadowsocks-all.sh 2>&1 | tee shadowsocks-all.lo
```

运行后会提示选择Shadowsocks server版本（有go、python、R、libev，我选的python版），输入端口号、密码、加密方式等等，可以选择默认值，一步步设置结束后，会输出配置信息、二维码、ss链接，最好复制粘贴出来方便后面客户端使用。

#### 2.2 查看配置

忘了的话，可以在/etc/shadowsocks-python目录下的config.json文件查看。

```shell
vi /etc/shadowsocks-python/config.json
```

### 3.网络加速

#### 3.1 BBR

```shell
wget –no-check-certificate https://github.com/teddysun/across/raw/master/bbr.sh && chmod +x bbr.sh && ./bbr.sh
```

这个比较耗内存，我在CentOS上经常内存溢出，不知道是不是这个脚本引起的

#### 3.2 Kcptun

这个加速器需要客户端也要安装client版本，所以有点麻烦。不过耗内存比较低。

##### 3.2.1服务端配置

```shell
wget --no-check-certificate https://github.com/kuoruan/shell-scripts/raw/master/kcptun/kcptun.sh
**chmod** +x ./kcptun.sh
./kcptun.sh
```

也需要设置 Kcptun 的服务端端口（可以随便设置一个没被占用的端口）、被加速的端口（上面配置的Shadowsock端口）、被加速IP（本机地址127.0.0.1）、密码，其他的可以默认。最后也会生成配置信息，也需要保存一下，后面配置客户端（window、手机）方便一些。

#####3.2.2查看配置

```shell
vi /usr/local/kcptun/server-config.json
```

#####3.2.3客户端配置

1.先到下载一个启动 Kcptun 的工具。请注意，这只是用来启动 Kcptun 的工具，而不是 Kcptun 客户端。

<https://github.com/dfdragon/kcptun_gclient/releases>

2.然后下载服务端对应版本的 Kcptun（保存下来的提示信息里有）

<https://github.com/xtaci/kcptun/releases>

如果没保存通过cat /var/log/kcptun/server.log查看：

```shell
root@v70758:/var/log/kcptun# cat server.log
2019/05/21 10:08:26 version: 20190515
2019/05/21 10:08:26 initiating key derivation
2019/05/21 10:08:26 listening on: [::]:29900
2019/05/21 10:08:26 target: 127.0.0.1:28256
2019/05/21 10:08:26 encryption: aes
2019/05/21 10:08:26 nodelay parameters: 0 30 2 1
2019/05/21 10:08:26 sndwnd: 512 rcvwnd: 512
2019/05/21 10:08:26 compression: true
2019/05/21 10:08:26 mtu: 1350
2019/05/21 10:08:26 datashard: 10 parityshard: 3
2019/05/21 10:08:26 acknodelay: false
2019/05/21 10:08:26 dscp: 0
2019/05/21 10:08:26 sockbuf: 4194304
2019/05/21 10:08:26 smuxbuf: 4194304
2019/05/21 10:08:26 keepalive: 10
2019/05/21 10:08:26 snmplog:
2019/05/21 10:08:26 snmpperiod: 60
2019/05/21 10:08:26 pprof: false
2019/05/21 10:08:26 quiet: false
```

当前是20190515版本

3.设置客户端

![图](https://jarrod-chen.github.io/img/post-20190521-kcptun.jpg)

本地监听端口自动生成就行不用管；KCP服务器地址是VPS的IP；端口是服务器kcptun的运行端口；可选参数如果要配置需要参照kcptun服务端的配置信息。

这样就整个梯子就搭好了。