### linux 下tcp链接状态

###### 命令:

```shell
 netstat -n
```

#### 三次握手

![img](20180717202520531)





#### 四次挥手

![img](20180717204202563)



## TCP状态解析

**ESTABLISHED:**

> **指TCP连接已建立，双方可以进行方向数据传递**

**CLOSE_WAIT:**

> **这种状态的含义其实是表示在等待关闭。当对方close一个SOCKET后发送FIN报文给自己，你系统毫无疑问地会回应一个ACK报文给对方，此时则进入到CLOSE_WAIT状态。接下来呢，实际上你真正需要考虑的事情是察看你是否还有数据发送给对方，如果没有的话， 那么你也就可以close 这个SOCKET，发送 FIN 报文给对方，也即关闭连接。所以你在CLOSE_WAIT 状态下，需要完成的事情是等待你去关闭连接。**

**LISTENING:**

> **指TCP正在监听端口，可以接受链接**

**TIME_WAIT:**

> **指连接已准备关闭。表示收到了对方的FIN报文，并发送出了ACK报文，就等2MSL后即可回到CLOSED可用状态了。如果FIN_WAIT_1状态下，收到了对方同时带FIN标志和ACK标志的报文时，可以直接进入到TIME_WAIT状态，而无须经过FIN_WAIT_2 状态。**

**FIN_WAIT_1:**

> **这个状态要好好解释一下，其实FIN_WAIT_1和 FIN_WAIT_2状态的真正含义都是表示等待对方的FIN报 文。而这两种状态的区别是：FIN_WAIT_1状态实际上是当SOCKET在ESTABLISHED状态时，它想主动关闭连接，向对方发送了FIN 报文，此时该SOCKET即进入到FIN_WAIT_1 状态。而当对方回应ACK 报文后，则进入到FIN_WAIT_2状态，当然在实际的正常情况 下，无论对方何种情况下，(被动关闭方, 接受到FIN的一方)都应该马上回应ACK报文，所以FIN_WAIT_1状态一般是比较难见到的，而FIN_WAIT_2 状态还有时常常可以用 netstat看到。**

**FIN_WAIT_2：**

> **上面已经详细解释了这种状态，实际上FIN_WAIT_2 状态下的SOCKET，表示半连接，也即有一方要求close 连接，但另外还告诉对方，我暂时还有点数据需要传送给你，稍后再关闭连接。**(主动关闭方接受到ACK, 但是没有收到FIN包. 说明被动 关闭方还有数据要发送)

**LAST_ACK:**

> **是被动关闭一方在发送FIN报文后，最后等待对方的ACK报文。当收到ACK报文后，也即可以进入到CLOSED可用状态了**

**SYNC_RECEIVED:**

> **收到对方的连接建立请求,这个状态表示接受到了SYN报文，在正常情况下，这个状态是服务器端的SOCKET在建立TCP连接时的三次握手会话过程中的一个中间状态，很短暂，基本上用netstat你是很难看到这种状态的，除非你特意写了一个客户端测试程序，故意将三次TCP握手过程中最后一个ACK报文不予发送。因此这种状态时，当收到客户端的ACK报文后，它会进入到ESTABLISHED状态。**

**SYNC_SEND:**

> **已经主动发出连接建立请求。与SYN_RCVD遥想呼应，当客户端SOCKET执行CONNECT连接时，它首先发送SYN报文，因此也随即它会进入到了SYN_SENT状态，并等待服务端的发送三次握手中的第2个报文。**



![img](https://pic1.zhimg.com/80/v2-0e93c846696208020702f1b8a60dff6c_720w.jpg)

用中文来描述下这个过程：

Client: `服务端大哥，我事情都干完了，准备撤了`，这里对应的就是客户端发了一个**FIN**

Server：`知道了，但是你等等我，我还要收收尾`，这里对应的就是服务端收到 **FIN** 后回应的 **ACK**

经过上面两步之后，服务端就会处于 **CLOSE_WAIT** 状态。过了一段时间 **Server** 收尾完了

Server：`小弟，哥哥我做完了，撤吧`，服务端发送了**FIN**

Client：`大哥，再见啊`，这里是客户端对服务端的一个 **ACK**

到此服务端就可以跑路了，但是客户端还不行。为什么呢？客户端还必须等待 **2MSL** 个时间，这里为什么客户端还不能直接跑路呢？主要是为了防止发送出去的 **ACK** 服务端没有收到，服务端重发 **FIN** 再次来询问，如果客户端发完就跑路了，那么服务端重发的时候就没人理他了。这个等待的时间长度也很讲究。



**Maximum Segment Lifetime** 报文最大生存时间，它是任何报文在网络上存在的最长时间，超过这个时间报文将被丢弃