---
title: 公司内部web服务器获取不到外网用户真实ip的解决方案
date: 2017-06-29 16:15:03
tags:
  - 网络
  - NAT
  - WinBox
category: tech
---

因为涉及到一些数据安全的问题，所以公司有些web项目是直接放在内网的。为了严格控制可以访问的用户，我们做了一套ip授权的机制。即，如果该ip没有被任何管理员登录过，那么用户在该ip就无法访问此项目。不过，最近这套机制，遇到了一些挑战。因为，通过我们nginx日志查看的时候，所有的客户端来源ip都只有一个——192.168.1.1，也就是我们的网关地址。很显然，不解决这个问题，我们的整套ip授权的机制就形同虚设了。

<!--more-->

## 方向一：真实ip通过额外的header头发送给web服务器了？

我们的公司出口ip是固定的，当用户在外网访问web项目时，路由器会根据既定的路由规则，将80端口的请求转发到内网一台web服务器的80端口。刚开始，我们怀疑，路由器对web请求做了一些篡改，应该会将原始ip通过`X-Forwarded-For`或者`X-Real-IP`这样的header头发送给web服务器，所以，我们使用nginx的`http_realip`模块进行了相应的处理。

```nginx
http {
    ...

    set_real_ip_from   192.168.1.1;
    real_ip_header     X-Forwarded-For;
    real_ip_header     X-Real-IP;

    ...
}
```

不过，结果不容乐观，客户端ip那里，还是很固执地显示网关的ip。为了进一步验证这一块，我们写了一个测试的php页面，把`$_SERVER`变量打印出来，丝毫看不到真实ip的影子。看来，这个方向不正确，得另寻出路了。

回过头来，其实路由器根本就不可能工作在网络协议的这么顶层，只会在ip包的层面上处理数据，没法深入到ip包内部篡改数据。

## 方向二：真实ip直接被替换掉了？

在应用层处理不了这个问题，那么只能深入到更顶端的硬件配置了，我们需要连上路由器，看看它的配置规则。我们的路由器没有直接的web管理界面，需要通过winbox连接。听这个名字，在mac系统的人可能不自觉地会一哆嗦，不会没有mac版吧。还好，这个软件有mac版。不过，也不是那么好安装的，这里花点笔墨说一下正确的安装过程。

### mac系统安装winbox

下载地址：[下载页面](http://joshaven.com/resources/tools/winbox-for-mac/)
     [最新版v3.11](http://joshaven.com/Winbox4Mac_3.11.dmg)
下载以后，有可能运行不了，出现错误提示：

```
The application X11 could not be opened.

An error occurred while starting the X11 server:
“Failed to activate core devices.”
Click Quit to quit X11. Click Report to see more details or send a report to Apple.
```

{% asset_img failed-to-activate-core-devices.jpg winbox无法运行%}

如果出现类似这样的界面，那么首先需要确保你安装了[X11 XQuartz 2.7.8](https://www.xquartz.org/)，这个软件用来帮助windows下的软件在mac环境下运行。

如果安装了这个软件以后，你还是不能运行WinBox，则需要考虑禁用 System Integrity Protection(SIP)功能了。一般地，在`Mac OS 10.11`以上的版本有可能遇到这个问题。`SIP`是一项增强的安全功能，锁定了`/System`、`/sbin`、`/usr`等目录。那么，怎么来禁用这项功能呢？

- 重启mac，电脑黑屏后，一直按着Command+R，进入恢复模式。进入到恢复模式的比平时进入系统要慢很多，大概需要等待3-5分钟。
- 进去以后，选择“实用工具”=>“终端”进入shell界面，输入"csrutil disable"，则`SIP`功能就被禁用了，然后重启系统即可。
  
  {% asset_img find-console.jpg 运行终端%}

### NAT的管理

网络地址转换（Network Address Translation，缩写为NAT），是一种在IP包通过路由器或防火墙时重写来源IP地址或目的IP地址的技术。这种技术被普遍使用在有多台主机，但只通过一个公网IP地址访问互联网的私有网络中。

打开WinBox，设定好路由器地址和帐号密码，连接好，通过 `IP` => `Firewall` => `NAT`，即可进入`NAT`的管理。

 {% asset_img nat-configure.jpg nat的管理%}

进去以后，这是配置列表：

 {% asset_img nat-interface.jpg nat的规则列表%}

其中，Chain为srcnat的，是对ip包的源地址进行进行修改，Chain为dstnat，则是对ip包的目标地址端口等进行修改。序号为0的规则就是srcnat的，它的Action为masquerade。也就是说，会把来源ip伪造成网关的地址，通过这样的篡改，使得局域网用户用一个共同的固定公网ip访问互联网。

不过，细想之下，这条规则虽然是用来给局域网用户共享上网设置的，但是，这条规则并没有任何过滤条件，比如，来源ip，或者ip包来自于哪个网卡。它既可以篡改从局域网出去的ip包，也可以控制通过互联网访问公司web服务器的ip包。也正是因为如此，当我们通过外网访问公司的web系统的时候，ip包的源地址替换成了网关的ip，导致在nginx的日志里边看不到这些请求的原始客户端IP。

想清楚了这一层，实际上要改动起来就很方便了，我们可以选择增加内网网卡的限制，还可以直接对来源ip做过滤即可。

{% asset_img srcnat-filter.jpg srcnat增加过滤选项%}

上图中，我们增加了`Src.Address`：192.168.0.0/16。然后点击右侧的"Apply"按钮即可保存设置。接下来，再打开nginx的访问日志，就会发现通过外网访问公司内部的web系统时的ip，已经变成了真实的客户端IP。不过，通过内网访问的请求，客户端ip还是网关的地址。不过，这已经不是特别要紧的事情了，我们已经能将外网用户的ip通过授权机制保证访问安全性了。

## 方向三：如何保证内网真实ip的正确显示

如果还想让内网用户的真实ip正确显示，可以通过dns劫持来篡改内网用户访问对应域名实际访问的ip，进入路径是`IP` => `DNS` => `Static`。

{% asset_img dns-configure.jpg DNS配置%}

当用户在内网访问公司的web系统时，由于路由器的dns劫持，将直接将请求发送到目标服务器上，因而能成功躲避srcnat规则对来源ip的篡改，最终让nginx成功获取到用户的内网真实ip。

## 小结

对于开发人员而言，要去解决网络方面的问题，还真的不是特别擅长。索性，互联网是开放的，我们可以通过一定的搜索技巧，经过学习、总结、归纳、分析，最终找到解决问题的正确方案。

参考链接：
  * <http://becomethesolution.com/blogs/mac/failed-to-activate-core-devices-x11-mac>
  * <http://bbs.feng.com/read-htm-tid-10015368.html>
  * <https://wiki.mikrotik.com/wiki/Manual:IP/Firewall/NAT>
  * <https://wenku.baidu.com/view/fab73ec358f5f61fb7366667.html>

