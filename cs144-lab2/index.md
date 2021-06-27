# CS144 lab2 详解


<!--more-->

> 实验指导说明：[https://cs144.github.io/assignments/lab2.pdf](https://cs144.github.io/assignments/lab2.pdf)

本实验目的为完成TCP的接收端，调用`segment_received()`接受TCP包(TCP segment)从而被前面两个lab中实现的`byte_stream`和`stream_reassembler`使用，同时调用`ackno()`和`window_size()`将`ack`和`win_size`返回给发送方，发送方接收到后可以确定接收方是否正确接受到上一个包，以及下一次发送包的大小范围。官方解释如下图：

![cs144-lab2-1](https://cdn.jsdelivr.net/gh/PeiLeiScott/image-hosting@master/cs144-lab2-1.3w91wydks940.png)


## `stream_index (64-bit)`和`seqno (32-bit)`之间的转换

![cs144-lab2-2](https://cdn.jsdelivr.net/gh/PeiLeiScott/image-hosting@master/cs144-lab2-2.5l1t56m3c6o0.jpg)

上图为TCP报文格式，其中红色区域为报头部分，共占20个字节，下方的`Payload`为数据部分。需要注意的是，在TCP包中`seqno`和`ackno`只占4个字节，也就是32-bit(原因在实验指导中提到了，有兴趣可以看看)。但是lab1中`stream_reassembler`写数据时`index`使用的是64-bit的数据，因此在向`stream_reassembler`写入数据以及向发送端返回`ack`之前需要完成相应的转换。

![cs144-lab2-3](https://cdn.jsdelivr.net/gh/PeiLeiScott/image-hosting@master/cs144-lab2-3.5gwqgmmqsxc0.jpg)

所有细节点都在上图中，`absolute seqno`和`stream index`之间的转换比较简单，只需要加一或减一即可，所以本实验我们只需要实现`seqno`和`absolute seqno`之间的转换。其中将`absolute seqno`转换为`seqno`(`wrap()函数`)比较简单，这里就不多说了，主要提一下`unwrap()`函数实现中的细节点。

`unwrap()`函数中有三个参数：序列号`n`、最初的序列号`isn (initial sequence number)`和`checkpoint`(不知道该怎么翻译。。。)
+ 第一步算出偏移量`offset`，比如`isn = 2^32 - 2, n = 1`的话，`offset = 1 - (2^32 - 2) = 3`，注意这里是32位无符号数的减法。
+ 算出`offset`后，`absolute seqno`并不是直接等于`offset`，而是有很多种可能情况，`offset`，`offset + 2^32`，`offset + 2 * 2^32`，`offset + 3 * 2^32`等等都有可能，这也是`checkpoint`存在的必要，我们需要从所有可能的情况中找到离`checkpoint`最近的那个数。
+ 接下来这步是最关键的一步。我们可以将`checkpoint`表示成`A * 2^32 + B`的形式，那么离`checkpoint`最近的数就只有下面三种情况：`(A-1) * 2^32 + offset`，`A * 2^32 + offset`，`(A+1) * 2^32 + offset`(不明白的话可以在纸上写一写)。因此我们要通过位运算求出A的值，然后判断下三个数哪个离`checkpoint`最近即可。需要注意的是，当A等于0时，`(A-1) * 2^32 + offset`是没有意义的，只需要考虑其他两个数就可以了。

代码实现如下：
~~~CPP
WrappingInt32 wrap(uint64_t n, WrappingInt32 isn) {
    // DUMMY_CODE(n, isn);
    return isn + static_cast<uint32_t>(n);
}

uint64_t unwrap(WrappingInt32 n, WrappingInt32 isn, uint64_t checkpoint) {
    // DUMMY_CODE(n, isn, checkpoint);
    uint32_t offset = n.raw_value() - isn.raw_value();
    uint64_t mask = static_cast<uint64_t>(UINT32_MAX) << 32;
    uint64_t top32 = checkpoint & mask;
    uint64_t num1 = (top32 - (1ul << 32)) + offset;
    uint64_t abs1 = checkpoint - num1;
    uint64_t num2 = top32 + offset;
    uint64_t abs2 = (checkpoint > num2) ? (checkpoint - num2) : (num2 - checkpoint);
    uint64_t num3 = (top32 + (1ul << 32)) + offset;
    uint64_t abs3 = num3 - checkpoint;
    if (top32 == 0) {
        return (abs2 < abs3) ? num2 : num3;
    }

    if (abs1 < abs2) {
        return (abs1 < abs3) ? num1 : num3;
    }

    return (abs2 < abs3) ? num2 : num3;
}
~~~


## 实现TCP接收端

### `segment_received()`
TCP接收端一共需要经过以下几个状态：

![cs144-lab2-4](https://cdn.jsdelivr.net/gh/PeiLeiScott/image-hosting@master/cs144-lab2-4.1yzga6hnjy00.jpg)

+ 最开始处于`listen`监听状态，等待收到发送方发来的`SYN`标志数据传输的开始。对应下方代码的 11-13 行。
+ 收到了`SYN`后进入第二个状态，将接收到的第一个`SYN`标志位为1的序列号作为`isn`，设置`_syn_recved`为true。第一次时不需要将`ack`转换为`stream_index`，因为此时`stream_index`就是0，直接调用`stream_reassembler`的`push_substring()`方法即可，但是在判断`eof`时需要注意第一次也有可能`FIN`标志位也为1。除了第一次之外，每次都需要计算`checkpoint`，然后调用`unwrap()`函数将`ackno`转换为`abs_ackno`，最后减一得到`stream_index`。
+ 收到`FIN`进入最后一个状态。设置`_fin_recved`和`eof`为true，标志数据输入完毕，如果从`tcp_receiver`到`stream_reassembler`和`byte_stream`之间传输没问题的话，`byte_stream`就应该调用`input_ended()`停止接受数据了。

代码实现如下：

~~~CPP
// tcp_receiver.hh
WrappingInt32 _isn{WrappingInt32(0)};
bool _syn_recved{false};
bool _fin_recved{false};

// tcp_receiver.cc
void TCPReceiver::segment_received(const TCPSegment &seg) {
    // DUMMY_CODE(seg);
    const TCPHeader header = seg.header();

    if (!_syn_recved && !header.syn) {
        return;
    }

    string data = seg.payload().copy();
    bool eof = false;

    // Receive the first SYN
    if (!_syn_recved && header.syn) {
        _syn_recved = true;
        _isn = header.seqno;
        // It's possible that the header have also FIN flag set.
        if (header.fin) {
            _fin_recved = true;
            eof = true;
        }
        _reassembler.push_substring(data, 0, eof);
        return;
    }

    // Receive the FIN
    if (_syn_recved && header.fin) {
        _fin_recved = true;
        eof = true;
    }

    uint64_t checkpoint = stream_out().bytes_written();
    uint64_t abs_seqno = unwrap(header.seqno, _isn, checkpoint);
    uint64_t stream_index = abs_seqno - 1;
    _reassembler.push_substring(data, stream_index, eof);
}
~~~

### `ackno()`

接收到数据后，`tcp_receiver`需要向发送方返回一个`ack`数据，发送方可以通过这个`ack`知道`tcp_receiver`是否成功收到上一个包，而这个`ack`就等于`tcp_receiver`下一步要接收的包的首字节。
+ 如果还没有接收到`SYN`的话，直接返回空数据。
+ 通过已经写入的字节数获取`abs_seqno`的值。比如在下图中，`bytes_written() = 3`，而`FIN`之前，也就是写入的最后的一个字节的`abs_seqno`就等于3，如果没有`FIN`的话，那么`ack`就等于`wrap(abs_seqno + 1, isn)`；有`FIN`的话，注意！！！只有当通过`FIN`标志得到的`eof`成功传到`stream_reassembler`和`byte_stream`后，`FIN`标志才真正生效了，也即此时`byte_stream`应该已经调用了`input_ended()`函数停止接收数据，这时候`abs_seqno`就不再是3而应该是4了。对应下方代码 11-13 行。

![cs144-lab2-3](https://cdn.jsdelivr.net/gh/PeiLeiScott/image-hosting@master/cs144-lab2-3.5gwqgmmqsxc0.jpg)

代码实现如下：
~~~CPP
optional<WrappingInt32> TCPReceiver::ackno() const {
    // Return empty if no SYN has been received
    if (!_syn_recved) {
        return nullopt;
    }

    uint64_t abs_seqno = stream_out().bytes_written();
    // FIN also accounts for one byte, but make sure that the reassembler
    // have successfully received the EOF signal and ended the input stream. 
    // Otherwise, something went wrong and The FIN means nothing.
    if (_fin_recved && stream_out().input_ended()) {
        abs_seqno++;
    }
    return optional<WrappingInt32>(wrap(abs_seqno + 1, _isn));
}
~~~


### `window_size()`

`window_size`标志着`tcp_receiver`还能容纳多少数据，从而告诉发送方下次发送数据的范围。通过lab0和lab1知道`byte_stream`和`stream_reassembler`其实是共用一个缓冲区的，因此剩余的空间直接调用`byte_stream`里的`remaining_capacity()`函数就可以了。


## 实验结果截图

![cs144-lab2-5](https://cdn.jsdelivr.net/gh/PeiLeiScott/image-hosting@master/cs144-lab2-5.4qr0wt6l1fo0.jpg)
