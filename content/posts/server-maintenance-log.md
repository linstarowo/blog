+++
date = '2026-11-13T22:43:11+08:00'
weather = "阴"
title = '方块相册维护日志(一)'
categories = [ "日志" ]
tags = [ "日志", "Minecraft" ]
+++

> ~~运维事故掩盖交流文档~~

## 关于方块相册

`方块相册` 一个Minecraft 1.20.1 Forge 服务器 隶属于PRcT社区服务器 从2023年暑假运行至今

~~广告~~: 服务器为长期档 轻度RP 养老建筑生存主题 正版验证 想玩可以加群 (QQ: 821657645)

我和厨师长(物理服主) 是整个服务器唯二的管理者 而我算是技术运维 负责大部分模组开放/维护 厨师长则负责服务器管理的大部分事务 而这个维护日志就用来记录一些我的Debug 踩坑经历 

## 起因

11 月 7 日是立冬 按照我们服务器的传统 客户端版本尊随节气的命名方式 所以我们就打算在下个星期更新一个小版本

内容并不多 主要是更新一些修复和优化模组 添加了一些人畜无害的新模组

这次更新最主要的部分 就是添加 `Reminia` 和 `Cocricot` 这两个移植模组 它俩是大名鼎鼎的 `Minia` 和 `Cocricot` 的高版本模组移植 提供了大量的精美建材 而问题恰好就出在这两个模组上

## 连接出现问题

当所有模组都被正确安装 服务器正常启动 玩家进入服务器 正在连接中... 通讯加密中... 加入世界中...

就在刚刚要进入加载地形的步骤时 连接突然中断 红色大字跃然屏幕

**您与Album 的连接出现问题**

查看服务器日志 没有任何报错 玩家握手成功后 过一会就被踢出

```
[Server] [17:14:59] [Server thread/INFO] [minecraft/ServerGamePacketListenerImpl]: PrimiaSUKI lost connection: Disconnected
```

再看客户端 同样没有任何错误 Sodium 准备加载世界之后就收到了来自服务器的 `ClientboundDisconnectPacket`

安装 `Spit it out` 模组 查看Debug 日志 全都一无所获 

通过~~简单高效的~~二分查找法 我们定位到问题就产生在 `Reminia` 和 `Cocricot` 这两个看似人畜无害的模组上 只要安装了这两个模组的 **任意一个** 就无法进入服务器

## 歧路

很显然 如果是代码层面的问题 两个模组都命中问题概率很小 最可能原因是 这两个模组都提供了大量的建筑方块 从而导致触碰到了一些可能的 **上限** 

但是服务器已经安装了 `PacketFixer` 扩大数据包大小 1.13 扁平化以后 方块ID 的注册限制也不复存在 最要命的是 在我们另一台机器的镜像服上 玩家可以正常进入服务器 这让我们做了很多错误的判断和尝试

卸载其他的建筑模组 —— 失败

清空地图重新生成 —— 失败

切换Jvm 关闭压缩对象头 —— 失败

更新Forge 版本 —— 失败

最后实在急眼了 我们直接把镜像服从另一台机器上拷贝到主服务器 厨师长暴力比较二者配置的不同之处(~~这点我是佩服他的~~) 最终我们终于定位到了根因

``` toml
# server.properties

network-compression-threshold=1048576
```

在我们进不去的主服务器 这个配置项的值是 **-1**

## 又是网络包!?

网络包问题就是这么神秘 没有任何报错 Debug 需要花老鼻子劲了 

Minecraft 使用 Netty 作为网络层 `network-compression-threshold` 这个参数会决定原版 Netty Pipeline 中 CompressionEncoder/Decoder 的压缩阈值 当设置为 -1 时 将不会把他们插入到管道中

初步推测 肯定是有一个雷霆大数据包导致的问题 但是实际上我们的数据大小上限已经被`Packet Fixer` 拓宽到了2 Gib 理论上是不会出现这样的问题的...

为了Debug 我直接写了个模组让服务器把所有收发的数据包名称都打印出来 但是每次截断到Disconnected 的数据包都是不一样的 但也并非一无所获

所有的其他的数据包 日志都是在主线程输出的 但是唯独 `ClientboundDisconnectPacket` 是在 `Netty IO` 线程发送的 我直接打印堆栈

```
[Server] [10:27:27] [Netty Server IO #4/INFO] [STDERR/]: [java.lang.Thread:dumpStack:2018]: java.lang.Exception: Stack trace
[Server] [10:27:27] [Netty Server IO #4/INFO] [STDERR/]: [java.lang.Thread:dumpStack:2018]:     at java.base/java.lang.Thread.dumpStack(Thread.java:2018)
[Server] [10:27:27] [Netty Server IO #4/INFO] [STDERR/]: [java.lang.Thread:dumpStack:2018]:     at TRANSFORMER/minecraft@1.20.1/net.minecraft.network.Connection.handler$dbc000$letmefix$logSend(Connection.java:2543)
[Server] [10:27:27] [Netty Server IO #4/INFO] [STDERR/]: [java.lang.Thread:dumpStack:2018]:     at TRANSFORMER/minecraft@1.20.1/net.minecraft.network.Connection.m_129520_(Connection.java)
[Server] [10:27:27] [Netty Server IO #4/INFO] [STDERR/]: [java.lang.Thread:dumpStack:2018]:     at TRANSFORMER/minecraft@1.20.1/net.minecraft.network.Connection.m_243124_(Connection.java:189)
[Server] [10:27:27] [Netty Server IO #4/INFO] [STDERR/]: [java.lang.Thread:dumpStack:2018]:     at TRANSFORMER/minecraft@1.20.1/net.minecraft.network.Connection.redirect$cnk000$fabric_networking_api_v1$resendOnExceptionCaught(Connection.java:2081)
[Server] [10:27:27] [Netty Server IO #4/INFO] [STDERR/]: [java.lang.Thread:dumpStack:2018]:     at TRANSFORMER/minecraft@1.20.1/net.minecraft.network.Connection.exceptionCaught(Connection.java:142)
[Server] [10:27:27] [Netty Server IO #4/INFO] [STDERR/]: [java.lang.Thread:dumpStack:2018]:     at MC-BOOTSTRAP/io.netty.transport@4.1.82.Final/io.netty.channel.AbstractChannelHandlerContext.invokeExceptionCaught(AbstractChannelHandlerContext.java:302)
[Server] [10:27:27] [Netty Server IO #4/INFO] [STDERR/]: [java.lang.Thread:dumpStack:2018]:     at MC-BOOTSTRAP/io.netty.transport@4.1.82.Final/io.netty.channel.AbstractChannelHandlerContext.invokeExceptionCaught(AbstractChannelHandlerContext.java:281)
```

调用来自 `Connection.m_243124_` 这方法的 SRV 名为 `exceptionCaught` 

Connection 这个类本身继承 Netty 的 `ChannelHandler` exceptionCaught 代表Netty 层抛出了异常

这里的异常之所以被吞掉和原版神秘的处理机制有关 也涉及到服务器安装了新雅互联 这里不多说 我直接手动输出了这个异常

```
[Server] [10:39:37] [Netty Server IO #2/ERROR] [minecraft/Connection]: Got Netty exception:
[Server] java.io.IOException: 你的主机中的软件中止了一个已建立的连接。
[Server]        at sun.nio.ch.SocketDispatcher.write0(Native Method) ~[?:?] {}
[Server]        at sun.nio.ch.SocketDispatcher.write(SocketDispatcher.java:57) ~[?:?] {}
[Server]        at sun.nio.ch.IOUtil.writeFromNativeBuffer(IOUtil.java:137) ~[?:?] {}
[Server]        at sun.nio.ch.IOUtil.write(IOUtil.java:81) ~[?:?] {}
[Server]        at sun.nio.ch.IOUtil.write(IOUtil.java:58) ~[?:?] {}
```

居然是写失败!? 但是很显然客户端收到了最后服务器因为这个 IOException 而发送给客户端的断开数据包并且正确断开了连接

所以服务器没出错 客户端没出错 答案只有一个

**Velocity!**

## Velocity 

`Velocity` 是一个现代化Proxy 转发 类似于BungeeCord 连接在服务器和客户端之间

所以是一个巨大的数据包 在服务器传递给客户端的过程中 直接阻塞了Velocity 让服务器收到了 RST 但实际上真正的客户端并没有断开 只是VC 发生了问题

关闭Velocity 让客户端直连服务器 问题果然解决

所以 到底是什么东西干的呢

我直接写了一个模组在日志里输出所有的数据包输出输出

```
// Server Side

[1411月2026 14:09:38.062] [Server thread/INFO] [net.minecraft.server.players.PlayerList/]: hail122[/115.200.98.206:61128] logged in with entity id 5591 at (2136.38481542024, 100.0, -1755.3128590008034)
[1411月2026 14:09:38.069] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundCustomPayloadPacket minecraft:register
[1411月2026 14:09:38.070] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundLoginPacket
[1411月2026 14:09:38.070] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundUpdateEnabledFeaturesPacket
[1411月2026 14:09:38.070] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundCustomPayloadPacket minecraft:brand
[1411月2026 14:09:38.070] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundChangeDifficultyPacket
[1411月2026 14:09:38.070] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundPlayerAbilitiesPacket
[1411月2026 14:09:38.070] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundSetCarriedItemPacket
[1411月2026 14:09:38.071] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundCustomPayloadPacket champions:main
[1411月2026 14:09:38.078] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundCustomPayloadPacket l2library:main
[1411月2026 14:09:38.078] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundCustomPayloadPacket l2library:main
[1411月2026 14:09:38.079] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundCustomPayloadPacket everyxhotpot:network
[1411月2026 14:09:38.079] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundCustomPayloadPacket ic_air:main
[1411月2026 14:09:38.080] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundCustomPayloadPacket ic_air:main
[1411月2026 14:09:38.082] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundCustomPayloadPacket kaleidoscope_tavern:network
[1411月2026 14:09:38.087] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundCustomPayloadPacket diet:main
[1411月2026 14:09:38.087] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundCustomPayloadPacket diet:main
[1411月2026 14:09:38.088] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundCustomPayloadPacket championsoptimization:main
[1411月2026 14:09:38.088] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundCustomPayloadPacket curios:main
[1411月2026 14:09:38.089] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundCustomPayloadPacket curios:main
[1411月2026 14:09:38.089] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundCustomPayloadPacket curios:main
[1411月2026 14:09:38.089] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundCustomPayloadPacket creativecore:main
[1411月2026 14:09:38.090] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundCustomPayloadPacket creativecore:main
[1411月2026 14:09:38.092] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundUpdateRecipesPacket
[1411月2026 14:09:38.117] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundUpdateTagsPacket
[1411月2026 14:09:38.117] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundEntityEventPacket
[1411月2026 14:09:38.163] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundCommandsPacket
[1411月2026 14:09:38.166] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundRecipePacket
[1411月2026 14:09:38.167] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundSetPlayerTeamPacket
[1411月2026 14:09:38.167] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundPlayerPositionPacket
[1411月2026 14:09:38.167] [Server thread/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] [S] -> ClientboundServerDataPacket

```

```
// Client Side

[1311月2026 22:43:40.863] [Netty Client IO #7/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] <- ClientboundLoginPacket
[1311月2026 22:43:40.863] [Netty Client IO #7/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] <- ClientboundClearTitlesPacket
[1311月2026 22:43:40.863] [Netty Client IO #7/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] <- ClientboundUpdateEnabledFeaturesPacket
[1311月2026 22:43:40.863] [Netty Client IO #7/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] <- ClientboundCustomPayloadPacket
[1311月2026 22:43:40.863] [Netty Client IO #7/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] <- ClientboundCustomPayloadPacket
[1311月2026 22:43:40.863] [Netty Client IO #7/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] <- ClientboundChangeDifficultyPacket
[1311月2026 22:43:40.863] [Netty Client IO #7/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] <- ClientboundPlayerAbilitiesPacket
[1311月2026 22:43:40.863] [Netty Client IO #7/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] <- ClientboundSetCarriedItemPacket
[1311月2026 22:43:40.863] [Netty Client IO #7/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] <- ClientboundCustomPayloadPacket
[1311月2026 22:43:40.863] [Netty Client IO #7/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] <- ClientboundCustomPayloadPacket
[1311月2026 22:43:40.864] [Netty Client IO #7/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] <- ClientboundCustomPayloadPacket
[1311月2026 22:43:40.864] [Netty Client IO #7/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] <- ClientboundCustomPayloadPacket
[1311月2026 22:43:40.864] [Netty Client IO #7/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] <- ClientboundCustomPayloadPacket
[1311月2026 22:43:40.864] [Netty Client IO #7/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] <- ClientboundCustomPayloadPacket
[1311月2026 22:43:40.864] [Netty Client IO #7/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] <- ClientboundCustomPayloadPacket
[1311月2026 22:43:40.865] [Netty Client IO #7/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] <- ClientboundCustomPayloadPacket
[1311月2026 22:43:40.887] [Render thread/INFO] [net.minecraft.client.multiplayer.ClientLevel/]: New ClientLevel instance created. there are 12 instances in heap
[1311月2026 22:43:40.892] [Render thread/INFO] [Oculus/]: Reloading pipeline on dimension change: NamespacedId{namespace='minecraft', name='overworld'} => NamespacedId{namespace='minecraft', name='overworld'}
[1311月2026 22:43:40.892] [Render thread/INFO] [Oculus/]: Destroying pipeline NamespacedId{namespace='minecraft', name='overworld'}
[1311月2026 22:43:40.897] [Netty Client IO #7/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] <- ClientboundCustomPayloadPacket
[1311月2026 22:43:40.900] [Netty Client IO #7/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] <- ClientboundCustomPayloadPacket
[1311月2026 22:43:40.900] [Netty Client IO #7/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] <- ClientboundCustomPayloadPacket
[1311月2026 22:43:40.900] [Netty Client IO #7/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] <- ClientboundCustomPayloadPacket
[1311月2026 22:43:40.900] [Netty Client IO #7/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] <- ClientboundCustomPayloadPacket
[1311月2026 22:43:40.900] [Netty Client IO #7/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] <- ClientboundCustomPayloadPacket
[1311月2026 22:43:40.900] [Netty Client IO #7/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] <- ClientboundCustomPayloadPacket
[1311月2026 22:43:40.903] [Render thread/INFO] [Oculus/]: Creating pipeline for dimension NamespacedId{namespace='minecraft', name='overworld'}
[1311月2026 22:43:40.904] [Render thread/INFO] [Oculus/]: Starting custom uniform resolving
[1311月2026 22:43:41.022] [Netty Client IO #7/INFO] [me.linstar.letmefix.LetMeFix/]: [Packet] <- ClientboundDisconnectPacket

```

可以看到 在一堆模组的 `CustomPayload` 包之后 (这些包主要是模组在onPlayerJoin 服务器完成玩家放置之后为了向客户端同步数据发送的) 服务器发送了 `ClientboundUpdateRecipesPacket` 这个包 但客户端迟迟没收到这个包 就连后续的数据包也被吞掉了 最终收到发来的 `ClientboundDisconnectPacket` 断开了连接

那么`ClientboundUpdateRecipesPacket` 这个包是干什么的呢 顾名思义 他会把服务器 **所有** 的配方都发送给客户端 在玩家加入和调用`/reload` 命令时发送 而`Reminia` 和 `Cocricot` 两个模组恰好都添加了大量的合成表 初步推测 这一个数据包可能得有好几个MB 那么大 因为光这两者的合成表数据加起来都高达**4MB**

删除两个模组的`/data/<id>/recipe` 果然能够进入服务器

## 尾言

问题解决了 心情非常的舒畅 但这个问题是在11 月 13 日被发现 11 月 14 日中午被解决

而我11.15 要期中考

绝望

~~写这篇文章的时候已经考完了 更绝望了~~




