## 《Netty实战》- ChannelHandler 和 ChannelContext

`ChannelHandlerContext` 代表了 `ChannelHandler` 和 `ChannelPipeline` 之间的关联，每当有 `ChannelHandler` 添加到 `ChannelPipeline`都会创建 `ChannelHandlerContext`。

#### 事件是如何流经pipeline的

![event stream](/Users/caoxiaoyong/Documents/blog/images/java/netty-channelhandlercontext.png)

因此在 `ChannelHandler` 的处理中，如果想把 event 传递给下一个 handler，就必须要显式的调用 `ChannelHandlerContext` 对应的等效方法，比如对于 in bound 必须要调用 `ctx.fireChannelXXX()`。

`ChannelPipeline` 上也有和 `ChannelHandlerContext`的同名方法，但是如果调用的是`ChannelPipeline`上的方法，event 会从 pipeline 的头部重新开始传播，比如下面的例子将形成一个环:

![pipeline](/Users/caoxiaoyong/Documents/blog/images/java/netty-channel-pipeline.png)

#### 资源管理

```
--msg--> handler1 --> handler2 --> handler3 --> ... --> handler n
```

假设在如上图所示的ChannelInboundHandler pipeline 中，如果想把 msg 传递下去，在每一个 handler 中都不要 release msg，确保 msg 始终是存活的。其实这是很简单的道理，但是我之前怎么在这里迷惑这么久呢？

#### 关于SimpleChannelInboundHandler

这个类有两点需要注意：

1. 如果想把 msg 传递给下一个 handler，需要在覆盖的 channelRead0(ctx, msg) 方法中显式的调用 `ctx.fireChannelRead(msg)`方法
2. 它的调用关系关系如下: channelRead --> channelRead0 --> ReferenceCountUtil.release(msg)，它会释放消息，如果想把 msg 传递给下一个 handler 的话，需要再 channelRead0(ctx, msg) 方法中显式的调用 `ReferenceCountUtil.retain(msg)`来确保 msg 不会被释放。

### 编解码

编解码的 handler 类中有一个原则：对于decode/encode方法的入参 msg 而言，我们的目的是要把它变成另外一种格式，完成编解码后原来的 msg 就没有用了，如果 msg 是 `ReferenceCounted` 类型都会调用它的`release()`方法，如果还需要的话记得调用它的`retain()`方法。

#### ByteToMessageDecoder extends ChannelInboundHandlerAdapter

这个类的处理流程可以总结如下：

1. 在`channelRead(ctx, Object msg)`中对于输入的ByteBuf in，将它 merge 到 `ByteBuf cumulation`字段上，同时释放输入的 in
2. 调用子类实现的`decode(ctx, cumulation, out)`，如果当前的数据足够解码出message，将message加入到out中
3. 如果out不为空，对于每个out的元素逐个调用ctx.fireChannelRead将消息发送出去

注意：这个类是 byte to message，所以只处理ByteBuf类型的输入，如果`in`的类型不是ByteBuf，则直接调用ctx.fireChannelRead(in)。

#### MessgeToMessgeDecoder`<I>` extends ChannelInboundHandlerAdapter

这个类是将`I`类型的message转成其他类型，代码逻辑如下：

```JAVA
@Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        CodecOutputList out = CodecOutputList.newInstance();
        if (msg is type of I) {
          I cast = (I) msg;
          // 调用用户实现的解码方法
          decode(ctx, cast, out);
          // 这里是将message转成另一种消息了，原来的消息就不需要了，所以需要release
          // 如果还需要msg，或者decode中得到仍然是msg，记得调用msg.retain()，否则就被释放了
          releaseIfReferenceCounted(cast);
        } else {
          // 不是本类需要处理的消息，转发给下一个handler来处理
          out.add(msg);
        }
        // 将消息发送给下一个handler
        for(Object targetMessage : out) {
            ctx.fireChannelRead(targetMessage);
        }
    }
```

#### MessgeToMessgeEncoder`<I>` extends ChannelOutboundHandlerAdapter

将`I`类型的message编码成其他类型，代码逻辑如下：

```java
@Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) 
        CodecOutputList out = CodecOutputList.newInstance();
        if (msg is type of I) {
          I cast = (I) msg;
          // 调用用户实现的编码方法
          encode(ctx, cast, out);
          // 这里是将message转成另一种消息了，原来的消息就不需要了，所以需要release
          // 如果还需要msg，或者encode中得到仍然是msg，记得调用msg.retain()，否则就被释放了
          releaseIfReferenceCounted(cast);
          // 至少要编码出一个Message
          asset(out is not empty);
          for(Object targetMessage : out) {
            write(targetMessage);
          }
        } else {
          // 不是本类需要处理的消息，转发给下一个handler来处理
          ctx.write(msg);
        }
    }
```

这个类和`MessgeToMessgeDecoder`很像，是它的反动作。但是也有一点不一致：这个类在写 target message 时有区别，如果输入的`msg`不是目标Message时，直接调用`ctx.write(msg)`来转发给下一个handler；而对于encode出的Message写入稍有区别。

#### MessgeToByteEncoder`<I>` extends ChannelOutboundHandlerAdapter

将`I`类型的message编码成ByteBuf，代码逻辑如下：

```java
@Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) 
        if (msg is type of I) {
          ByteBuf buf = allocate byte buffer;
          I cast = (I) msg;
          // 调用用户实现的编码方法，将 msg 写入到 buf中
          encode(ctx, cast, buf);
          // 这里是将message转成另一种消息了，原来的消息就不需要了，所以需要release
          // 如果还需要msg，记得调用msg.retain()，否则就被释放了
          releaseIfReferenceCounted(cast);
          
          // 发送
          ctx.write(buf, promise);
        } else {
          // 不是本类需要处理的消息，转发给下一个handler来处理
          ctx.write(msg, promise);
        }
    }
```

