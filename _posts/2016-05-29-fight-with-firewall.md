---
layout: post
title: 与防火墙的抗争
excerpt: 公司的机房对于安全特别较真，端口不轻易开放给开发者，想做远程调试非常费劲。这个文章记录我如何与防火墙抗争，最后赢得主动权的。真不容易。
modified: 2016-05-29
tags: [java, 防火墙, 远程调试, ssh tunnel, ssh隧道, socat]
comments: true
categories: tech
---

## 远程调试
用过Java的人都知道，jvm的远程调试可谓开发者的利器，线上问题的排查离不开它。常用如下的命令来启动
```
java -Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=y,address=19000 mainclass
```
具体的参数大家可以去查一下，最常用的是`suspend`。如果`suspend=y`，那么jvm启动起来后什么都不加载，知道有debug连接建立。如果是程序启动过程中出现了问题，就要用这个参数。

由于公司对于机房数据的安全要求特别严格，防火墙限制地特别死，办公室无法直接连接到机房，必须通过SSH到堡垒机登陆。
这样就没法建立远程连接来启用jvm的远程调试。怎么办呢？

## SSH隧道
SSH隧道可用于翻墙，同样可以用来帮助我‘翻墙’到机房内建立远程调试的连接。做法如下：

1. 假设运行服务的server为S，堡垒机跳进去后的机器为A，本地开发机为D。
2. 在S上启动服务，使用参数
```
java -Xdebug -Xrunjdwp:transport=dt_socket,server=y,address=12345 -jar myproduct-jar-with-dependencies.jar &> console.out &
```
3. 在D上登录堡垒机，使用命令`ssh -L 9000:S.ip:12345 songhou@baolei.ip`，然后登录到机器A。需要确保A能连接到S的12345端口。
4. 在D上打开Eclipse，新建remote debug，如下图：
![eclipse 1](/assets/images/Firewalls1.png)
![eclipse 2](/assets/images/Firewalls2.png)
然后点debug就可以连接到S上面的服务进程，在D的代码里面设置断点，然后调用在S上面的服务，就可以在D上调试代码了。

然后世界太平了？图森破。

## 端口转发
上面的方法有个前提，就是堡垒机上的sshd必须启用了ssh tunnel。
新机房启用后，采用了新的堡垒机提供商，机器的管理更加方便一点，但是这个堡垒机把ssh tunnel给禁用了。不能向它屈服。
既然只是需要一个能连上的端口，其实我并不需要完整的tunnel。一个临时的端口转发就可以了，工具`socat`就可以做到。

找到管安全的同事，将一个机器的一个固定端口开放给办公室可以直连，而这个机器跟我所有的其他的机器在同一个区内，可以直连。我就用它作为中间机器来实现端口转发。

1. 假设这个中间机器的IP是A，开放出来的端口是port_a。需要调试的程序位于S，端口是port_b。
2. 在A上运行命令：
```
socat tcp-listen:port_a,reuseaddr,fork tcp:S:port_b
```
3. 如前文类似，在本地启动调试程序，不过需要连接的remote地址是A:port_a。

## 关于JMX
JMX连接无法直接用前文描述的方法，因为它不只是有一个TCP连接，而是类似于FTP一样，建立连接后会使用另外的一个端口建立数据连接。
但是我们有类似的方法能做到，参加[stack overflow里的回答](http://stackoverflow.com/questions/15093376/jconsole-over-ssh-local-port-forwarding) 。
