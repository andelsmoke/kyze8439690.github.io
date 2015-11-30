---
layout: post
title: "Terminal 代理方案"
date: 2015-07-19 22:03:31 +0800
comments: true
categories: 
---

由于墙越建越高，现在想要访问国外的服务越来越难了。在购买了自己的VPS之后，可以基于各种方案建立对应的翻墙方案：

<!-- more -->

- Windows/Linux/Mac/Android: shadowsocks ( PAC )
- iOS: AnyConnect ( 路由表 )

~~Openvpn 由于特征过于明显已经不能使用。~~

以上方案暂时能够满足各种Desktop环境的需求，但是在使用Terminal的时候，会发现以上方案并不好用。这时候就要祭出大杀器 [ProxyChains-ng](https://github.com/rofl0r/proxychains-ng) 了。

安装好 proxychains4 之后，建立默认配置，指向本地 shadowsocks 建立的socks5代理。

```
# /etc/proxychains.conf
strict_chain

proxy_dns

remote_dns_subnet 224

tcp_read_time_out 15000
tcp_connect_time_out 8000

[ProxyList]
socks5  127.0.0.1 1080 #define your local socks proxy here
```

这时可以测试你的代理有没有生效了

```
➜  cnbeta git:(develop) proxychains4 git pull origin develop
[proxychains] config file found: /etc/proxychains.conf
[proxychains] preloading /usr/lib/libproxychains4.dylib
[proxychains] DLL init: proxychains-ng 4.8.1-git-4-gea51cda
[proxychains] DLL init: proxychains-ng 4.8.1-git-4-gea51cda
[proxychains] DLL init: proxychains-ng 4.8.1-git-4-gea51cda
[proxychains] DLL init: proxychains-ng 4.8.1-git-4-gea51cda
[proxychains] DLL init: proxychains-ng 4.8.1-git-4-gea51cda
[proxychains] DLL init: proxychains-ng 4.8.1-git-4-gea51cda
[proxychains] DLL init: proxychains-ng 4.8.1-git-4-gea51cda
[proxychains] DLL init: proxychains-ng 4.8.1-git-4-gea51cda
[proxychains] DLL init: proxychains-ng 4.8.1-git-4-gea51cda
[proxychains] DLL init: proxychains-ng 4.8.1-git-4-gea51cda
[proxychains] DLL init: proxychains-ng 4.8.1-git-4-gea51cda
[proxychains] DLL init: proxychains-ng 4.8.1-git-4-gea51cda
[proxychains] DLL init: proxychains-ng 4.8.1-git-4-gea51cda
[proxychains] DLL init: proxychains-ng 4.8.1-git-4-gea51cda
[proxychains] DLL init: proxychains-ng 4.8.1-git-4-gea51cda
[proxychains] DLL init: proxychains-ng 4.8.1-git-4-gea51cda
[proxychains] DLL init: proxychains-ng 4.8.1-git-4-gea51cda
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  github.com:22  ...  OK
[proxychains] DLL init: proxychains-ng 4.8.1-git-4-gea51cda
From github.com:kyze8439690/cnbeta
 * branch            develop    -> FETCH_HEAD
[proxychains] DLL init: proxychains-ng 4.8.1-git-4-gea51cda
[proxychains] DLL init: proxychains-ng 4.8.1-git-4-gea51cda
[proxychains] DLL init: proxychains-ng 4.8.1-git-4-gea51cda
[proxychains] DLL init: proxychains-ng 4.8.1-git-4-gea51cda
[proxychains] DLL init: proxychains-ng 4.8.1-git-4-gea51cda
[proxychains] DLL init: proxychains-ng 4.8.1-git-4-gea51cda
Already up-to-date.
```

如果看到一堆 [proxychains] 就说明代理成功了，以后想要运行的命令通过代理访问网络，就在运行的命令前增加 "proxychains4 "前缀即可（注意空格）

当然 proxychains4 可能太长，加个alias:

```
# ~/.zshrc
alias pc='proxychains4'
```

当然 ```pc git pull``` 还是太麻烦，而且会丢失补全的功能，如果你使用的是 Mac + iTerm 的组合的话，可以在 iTerm -> Preferences -> Profiles -> Keys 中，新建一个快捷键，例如 ```⌥+↩︎``` ，Action 选择 ```Send Hex Code```，键值为 ```0x1 0x70 0x63 0x20 0xd```，保存生效。

这样的话，以后命令要代理就直接敲命令，然后 ```⌥+↩︎``` 即可，这样命令补全也能保留了。

附上 Hex Code 对应表，获取工具为 [keycodes](http://manytricks.com/keycodes/)

Hex Code  | Key       |
----------|-----------|
0x1       | ⌃ + a     |
0x70      |  p        |
0x63      |  c        |
0x20      |  [space]  |
0xd       |  ↩︎       |  
  
  
**2015/11/30 更新**

由于 OSX 10.11 的 SIP 特性，会导致 proxychains-ng 安装失败，这里有两种解决方法：

- 如果是使用 brew install proxychains-ng 安装的话，由于没有写入权限，必须暂时关闭 SIP，安装成功之后再打开 SIP。具体方法见 [http://osxdaily.com/2015/10/05/disable-rootless-system-integrity-protection-mac-os-x/](http://osxdaily.com/2015/10/05/disable-rootless-system-integrity-protection-mac-os-x/)
- 如果不使用 brew install 的话，可以 clone 源码自己编译安装，关键是避免安装到 usr 目录（无法写入），手动指定写入目录，如 `./configure --prefix=$HOME/.local --sysconfdir=/etc`，etc 有写入权限不必修改，记得添加环境变量即可

初此之外，OSX 自带的 git，curl 等版本过低，无法支持 proxychains-ng，请手动更新版本。

**Thanks to:**

- [http://everet.org](http://everet.org)
- [Chenye](http://weibo.com/210106468)