# 延迟、抖动和可靠性

## 延迟

### 非网络延迟

- **输入采样延迟**：一般从用户按下一个按钮到游戏检测到这个按钮被按下，大概有半帧的时间

- **渲染流水线延迟**：如果有很多渲染任务要做，GPU 给用户显示渲染图像可能会滞后 CPU 一帧的时间。

- **多线程渲染流水线延迟**： 模拟线程准备模拟下一帧的时候，渲染线程批量处理 GPU 请求。

- **垂直同步**：垂直同步其实就是只在显示器的垂直消隐间隙改变视频卡显示的图像。但哪怕当前帧超过显示器刷新帧率 1ms ， 都会造成长达 1 帧的延迟。

- **显示延迟**：LCD 或者 HDTV 这些会对视频图像进行后处理时出现的代价。

- **像素响应时间**：LCD 这些显示器在更新像素的时候会出现重影。

### 网络延迟

- 处理延迟，传输延迟，排队延迟，传播延迟。
- RTT 是 **往返时间** 指的是从一台主机传输到另一台主机的时间，加上响应数据包返回的时间。

## 抖动

- 因为 RTT 其实不是一个常数，RTT 与平均延迟（期望） 的偏差就叫做抖动。
  - 处理延迟，网络延迟的最小部分，也是对抖动给你贡献最小。（一般发生在数据包路线发生调整时）
  - 传输延迟和传播延迟（TD 和 PD）链路层协议决定了传输延迟，路由长度决定了传播延迟。
  - 当包量比较多的时候也会有排队延迟的变化。

- 抖动会影响 RTT 抑制算法，也会导致数据包乱序到达。要解决包乱序，需要包重组或者用 TCP 那样的可靠传输机制。

- 避免抖动的方法：
  - 发送尽量少的包来保持低流量，优化服务器集群的位置，合理将处理任务拆帧，避免帧率导致的抖动。

## 数据包丢失

- 数据包丢失的三个主要成因：不可靠的物理介质， 不可靠的链路层（信道满，丢失正在发送的帧），不可靠的网络层，处理数据包的速度低与到达路由器的速度，导致数据包队列满了。

- 大部分路由器的处理能力是以数据包的个数为基础的，而不是总数据量，有时候发送一个大包回避发送多个小包有更好的性能收益。

## TCP 和 UDP

### TCP

- TCP 发送的东西都要求必须被按序处理。
  - 低优先级数据的丢失干扰高优先级数据的接收。
  - 两个单独的可靠有序数据流互相干扰。
  - 过时的游戏状态重传。
  - TCP Nagle算法会导致数据包传输速度变慢。
  - 操作系统必须保存所以偶的发送数据副本，直到数据被确认。

## 数据包传递通知

- 可靠系统的基础是**有能力知道数据包是否到达目的地**。
  - 传递通知模块的任务是帮助高层依赖模块发送数据包，并通知上层模块数据包是否送到。本身不实现重传。
  - 传递通知模块 **DeliveryNotificationManaager：**
    - 传输端，唯一标识每个传出的数据包，绑定传递状态和数据包
    - 接收端，检查传入包，上报状态并回馈给发送端。
    - 传输端同时要处理 数据包的确认包，通知上层依赖组件数据包的接收情况（哪些被接收，哪些被丢弃）
  - 在确认数据包是否到达目的的的的同时，也保证了数据包不会乱序到达，如果旧数据包在新数据包之后到达，DeliveryNotificationManager 会假装这个数据包被丢弃。

### 标记传出的数据包

- UDP 中的序列号不和 TCP 中一样，无需标识流中的字节数，只需要简单为每个传输数据包提供唯一标识符。
- 为了使用 DeliveryNotificationManager 传输数据，需要创建 OutMemoryBitStream，传入 DeliveryNotificationManager::WriteSequenceNumber（）的方法中。

```c++
InFlightPacket* DeliveryNotificationManager::WriteSequenceNumber(OutputMemoryBitStream& inPacket)
{
    PacketSequenceNumber sequenceNumber = mNextOutgoingSequenceNumber++;
    inPacket.Write(sequenceNumber);

    ++mDispatchedPacketCount;

    mInFlightPackets.emplace_back(sequenceNumber);
    return &mInFlightPackets.back();
}
```

- `mInFlightPackets` 负责存储那些发出而尚未确认的包。
- 标记之后应用程序写入数据包负载并发送给目的主机。

### 接收数据包并发送确认

```C++
/**
*将问题可以分成三类
*/
bool DeliveryNotificationManager::ProcessSequenceNumber(InputMemoryBitStream& inPacket)
{
  PacketSequenceNumber sequenceNumber;

  inPacket.Read(sequenceNumber);
  if(sequenceNumber == mNextExpectedSequenceNumber)
  {
    //如果这是希望那个帧。
    mNextExpectedSequenceNumber = sequenceNumber + 1;
    //这里先把sequenceNumber 缓存到队列中
    AddPendingAck(sequenceNumber);
    return true;
  }else if(sequenceNumber < mNextExpectedSequenceNumer)
  {
    //出现了过期帧，直接丢弃
    return false;
  }else if(sequenceNumber > mNextExpectedSequenceNumber)
  {
    //将当前帧往后移，这个时候会将所有跳过的帧
    mNextExpectedSequenceNumber = sequenceNumer +1;
    //如果发送端侦测到 ack 的中断，它可以重发丢失的包
    AddPendingAck(sequenceNumber);
    return true;
  }
}
```

```C++
void DeliveryNotificationManager::AddPendingAck(PacketSequenceNumber inSequenceNumber)
{
    if(mPendingAcks.size()==0
    ||!mPendingAcks.back().ExtendIfBound(inSequenceNumber))
    {
        mPendingAcks.emplace_back(inSequenceNumber);
    }
}
```

- `ACKRange` 表示要确认的连续序列号的集合。 `mStart` 成员变量存储第一个要确认的序列号，`mCount`成员变量记录要确认的序列号的数量。

```C++

inline bool AckRange::ExtendIfBound(PacketSequenceNumber inSequenceNumber)
{
    if(inSequenceNumber == mStart + mCount)
    {
        ++mCount;
        return true;
    }else
    {
        return false;
    }
}

void AckRange::Write(OutputMemoryBitStream& inPacket) const
{
}

void AckRange::Read(InMemoryBitStream& inPacket)
{
    
}

```

- `ExtendIfShould()` 方法检查序列号是否是连续的。如果是，增加计数，并告诉调用者范围扩大了。如果不是，返回错误，这样调用者便知道为不连续的序列号构建一个新的 `AckRange`.
- `Write()` 和 `Read()` 方法的工作方式是先序列化开始序列号，再序列化个数。