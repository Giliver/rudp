这个rudp做了一些tcp要做的实现，但是也看去了tcp的部分功能：
1. 内部完成重传机制，保存当前接受到的消息窗口
2. 允许丢包，超过一定时间的包即使对端没有收到也直接丢弃
3. 内部重传要求message有msg_id,
4. 允许超时丢包功能则要求message有tick_id.
5. 没有窗口大小协商机制，也没有拥塞控制等， 用户层控制具体联网发送策略。

<br/>

整个用户态逻辑是：
1. 将消息塞到发送队列
2. 从接收队列拿到消息
3. 接收udp数据，更新时间片，发送udp数据
4. 当然还要实际发送数据，通过socket/udp


<br/>

在更新时间片这个逻辑里面:
1. 解析接受udp数据，包括需要重传什么包，那些包已经丢了，数据包等
2. 重传包请求则查看发送历史时候有对应的包
3. 包丢失的话直接返回给应用层
4. 数据包的话直接塞到接受队列
5. 看发送队列有什么数据
6. 打包成多个rudp_package, 每个package的大小：
   - 如果没有特殊限制是128/256/512这样MTU大小
   - 如果逻辑包比较大，直接使用逻辑包的大小

<br/>

内部发送单元格式是
- 1-2字节的tag(包括大小) +
- 2字节的message id +
- 实际数据

因为message id只有2个字节，所以需要考虑回绕问题，这段代码是
- 首先将id和recv_id_max 高16位设置相同
- 如果recv_id_max比较大而id很小, (recv_id_max - 0x8000) > id, 说明id出现了一次回绕，需要+0x10000
- 如果recv_id_maxB比较小而id很大，(recv_id_max + 0x8000) < id, 说明recv_id_max有一次回绕（进位）而id没有跟上，所以需要-0x10000

```c
static int get_id(struct rudp *U, const uint8_t *buffer) {
    int id = buffer[0] * 256 + buffer[1];
    id |= U->recv_id_max & ~0xffff;
    if (id < U->recv_id_max - 0x8000)
        id += 0x10000;
    else if (id > U->recv_id_max + 0x8000)
        id -= 0x10000;
    return id;
}
```
