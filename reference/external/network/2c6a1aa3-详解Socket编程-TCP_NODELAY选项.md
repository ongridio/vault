---
title: 详解Socket编程---TCP_NODELAY选项
source: https://www.jianshu.com/p/ccafdeda0b95
kind: external
domain: network
original_date: 2018-07-22
fetched_at: 2026-05-16
bookmark_title: 详解Socket编程---TCP_NODELAY选项 - 简书
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.jianshu.com](https://www.jianshu.com/p/ccafdeda0b95)
> 原始日期：2018-07-22
> 抓取日期：2026-05-16

# 详解Socket编程---TCP_NODELAY选项

#### Nagle算法描述

Socket编程中，TCP_NODELAY选项是用来控制是否开启Nagle算法，该算法是为了提高较慢的广域网传输效率，减小小分组的报文个数，完整描述：

该算法要求一个TCP连接上最多只能有一个未被确认的小分组，在该小分组的确认到来之前，不能发送其他小分组。


这里的小分组指的是报文长度小于MSS(Max Segment Size)长度的分组（MSS是在TCP握手的时候在报文选项里面进行通告的大小，主要是用来限制另一端发送数据的长度，防止IP数据包被分段，提高效率，一般是链路层的传输最大传输单元大小减去IP首部与TCP首部大小）。

如果小分组的确认ACK一直没有回来，那么就可能会触发TCP超时重传的定时器。

下面是一个简单的示意图，开启了Nagle算法与没有开启：

#### 抓包分析

##### 默认开启Nagle算法

由于局域网内延迟低，不容易看到开启Nagle算法的效果，所以专门整个腾讯云的服务器测试，延迟在40毫秒左右。

Java代码与Unix C的Socket接口类似，这里使用Java代码作为示例简单一点。默认情况下Nagle算法是开启的，即socket.getTcpNoDelay()返回的数值为false，我们先分析这种场景。

Receiver的代码：

```
try (ServerSocket serverSocket = new ServerSocket()) {
serverSocket.bind(new InetSocketAddress(10086));//wildcard ip
Socket socket = serverSocket.accept();
System.out.println("Accept New Socket");
System.out.println("Tcp No Delay : " + socket.getTcpNoDelay());
InputStream is = socket.getInputStream();
OutputStream os = socket.getOutputStream();
int result;
while((result = is.read()) != -1) {
System.out.println((char)result);
}
TimeUnit.MINUTES.sleep(1);
}
```


Sender的代码：

```
try(Socket socket = new Socket()) {
socket.connect(new InetSocketAddress("212.64.20.XX", 10086));
System.out.println("Tcp No Delay : " + socket.getTcpNoDelay());
InputStream is = socket.getInputStream();
OutputStream os = socket.getOutputStream();
for (byte c : "TCP_NO_DELAY".getBytes()) {
TimeUnit.MILLISECONDS.sleep(10);
os.write(c);
os.flush();
}
TimeUnit.MINUTES.sleep(1);
} catch (IOException e) {
e.printStackTrace();
}
```


与服务器的延时在40毫秒左右，所以Sender这里每隔10毫秒就发送一次就可以演示出累计的小分组在收到ACK后才发送。注意如果是TCP发送的数据延迟还包含链路来回的延迟与Receiver捎带确认的延迟。

这里抓包工具使用的是tcpdump，导出pcap文件后再使用wireshark观察发送与接收数据的过程。

```
$ sudo tcpdump -v port 10086 -w TCP_DELAY.pcap
```


首先第一行到第三行是TCP三次握手的报文，可以看到双方都各自通告的MSS大小，发送端的报文小于这个大小就可以理解为小分组

第四行是Sender向Receiver发送的第一个字符'T'，对应的Len=1

第五行是Receiver回来对第四行发送消息的确认ACK

第六行，前面使用ping测试的延迟在40毫秒左右，而我们每10毫秒就会一个字符写到OS维护的发送缓冲区，所以确认ACK回来后，就已经累计了4个字符"CP_N"，发送的数据就是这4个字符

之后的流程和上面的类似，可能会出现发送不是4个字符的情况，出现的原因就是延迟可能小于或者大于40毫秒


下面是使用wireshark导出的时序图帮助进一步帮助理解这个流程。

##### 关闭Nagle算法

只需要在发送数据之前对Socket调用一个简单的方法就可以关闭Nagle算法：

```
socket.setTcpNoDelay(true);
```


直接抓包，看下报文：

可以看到，在Sender每10毫秒发送一个字符，不需要等到Receiver发送确认ACK，就继续发送，没有将数据放到OS维护的缓冲区。

下面是使用wireshark导出的时序图：

#### 总结

这个选项应该根据适合的场景进行判断关闭与否，例如实时性要求比较高的场景，类似用户鼠标操作，键盘输入，触摸屏事件输入，状态更新等这种连续的小分组数据，需要在对端立刻呈现，让用户尽可能感受不到延迟。但是如果网络延迟比较高，采用这种方式，那么会导致网路利用率下降。

一般类似HTTP协议请求响应的模型的场景不太需要考虑禁用这个算法，因为在一条TCP连接上发送小报文，不管多小都代表了服务端任务执行的指示，完成了这个请求之后才能继续执行下一个请求，即使Sender端提前发送过去也没有作用，所以开启Nagle算法是能够优化网络传输的，并且在Receiver端有捎带延迟确认，省掉单独的ACK确认进一步优化小分组传输。

另外HTTP2与HTTP协议不同，HTTP2是在一条TCP连接上进行所有HTTP请求，并且请求头部是压缩的就进一步加大了请求小分组的可能性，多个小分组HTTP请求并且分组大小的和小于MSS就会导致有延迟的现象，所以HTTP2的实现TCP_NODELAY选项是默认开启的。关于这点可以参考HTTP2对TCP_NODELAY的描述。