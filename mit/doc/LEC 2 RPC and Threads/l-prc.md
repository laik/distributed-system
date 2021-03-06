# RPC目标

rpc是分布式系统的基础，rpc的目标是：

- 简单的网络通信
- 将客户端/服务端的通信细节隐藏起来
- 客户端调用就像普通的过程调用
- 服务器处理程序就像普通程序

使用rpc后，网络通信就像下面这样子：

```
Client:
    z = fn(x, y)
Server:
    fn(x, y) {
      compute
      return z
    }
```

目标就是做到网络通信对应用透明

rpc消息的流程图

```
Client             Server
    request--->
       <---response
```

软件架构

```
		client app         handlers
    	stubs           dispatcher
   		RPC lib           RPC lib
     　　net  ------------ net
```

细节

客户端怎么知道应该跟谁通信？

​	客户端提供服务端的地址

​	由命名服务来提供服务端名字到服务器地址的映射

怎么处理rpc失败？

​	失败的原因可能是：丢包，网络断线，服务器运行缓慢，服务器崩溃

交互失败对rpc的client端意味着什么？

​	client端收不到server端的响应

​	client端不知道请求是否发送到了server端，server端是否执行了

那怎么处理失败的情况呢？简单的方案是使用“至少一次”语义

​	rpc等待一段时间的响应，如果没有响应到达，重新发送请求，重复该过程一定次数直到请求返回，如果这个时候，还是没收到响应，则返回应用一个错误

“至少一次”的语义应用在处理上是否简单呢？

​	在写请求上，以"deduct $10 from bank account"为例，可能会多次扣款

​	从客户端角度，先后发送两个请求：Put("k", 10)，Put("k", 20)，但是由于网络的原因，第一个请求比第二个请求慢到达server端，出现数据和我们预期的不一样的问题

那“至少一次”的语义有什么场景下是合适的呢？

​	如果操作是可重复的，譬如：读请求

​	如果像GFS那样自己能处理多副本的

那除了“至少一次”的语义外，还有没有更好的处理方式？"at most once"至多一次

​	理想情况是server端能识别重复请求，对于重复请求返回之前的处理结果，那怎么察觉是重复请求呢？

​	client端可以对每个请求都带上唯一的ID，server端根据ID来处理：

```
	if seen[xid]:
      r = old[xid]
    else
      r = handler()
      old[xid] = r
      seen[xid] = true
```

那至多一次的语义会带来什么样的复杂性呢？

第一个怎么保证ID是全局唯一的？

server端保存old[xid]信息和seen[xid]信息保存到什么时候？

​	理想情况下：每个客户端都有唯一的id，每个客户端rpc请求都有seq号，客户端发送请求的时候，会带着我已经看到所有的<=X的请求了，类似于TCP的sequ和ack机制

​	或者只允许客户端一次发一个请求，当seq+1的请求到达客户端，小于seq的信息就可以丢弃了

​	或者客户端和服务器端达成共识：只重试**<5**分钟的请求，那服务器端就丢弃大于5分钟之前的信息

考虑另一个问题：如果重复请求过来的时候，之前的请求还在处理中怎么办？这个时候不能重复执行，好的办法是在rpc中加入“pending”标志或者等待或者忽略



如果一个最多一次语义的服务端挂了或者重启了怎么办？

如果用于至多一次的语义的信息是存储在内存中的，那重启后信息就丢失了，会接受重复的请求，因此至多一次的语义的信息保存在硬盘上会比较好，或者做冗余存储，多副本。



第三种方案：精确执行一次"exactly once"

实现上就是最多一次的方案+无限重试+容错服务



Go RPC实现的”最多一次“

- 打开TCP连接
- 向TCP连接写入请求
- TCP可能会重传，但是服务器的TCP协议栈会过滤重复的信息
- 在Go代码里面不会有重试（即：不会创建第二个TCP连接）
- Go RPC代码当没有获取到响应之后将返回错误
  - 也许是TCP连接的超时
  - 也许是服务器没有看到请求
  - 也许服务器处理了请求，但是在返回回复之前服务器的网络故障



