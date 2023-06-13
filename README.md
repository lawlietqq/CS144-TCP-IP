lab 地址：https://cs144.github.io/

lab 目的：实现一个较为完整的TCP协议

Lab0 字节流：
类似于通道，实现字节流的基本功能，包括复制、读、写等功能。
代码较简单，不放截图，实验结果如下所示：

<img width="412" alt="image" src="https://github.com/lawlietqq/CS144-TCP-IP/assets/92260319/64d54151-7408-45aa-81cf-04d74896d520">

Lab1 流重组器 StreamReassembler
lab1的概念图如下所示：

<img width="416" alt="image" src="https://github.com/lawlietqq/CS144-TCP-IP/assets/92260319/af556909-ec69-4492-9f7f-b6c4a5531e6a">

本lab实现流重组器，负责在不可信的网络环境中实现可信传输，序号与子串一一对应，重组完成后将子串输入到ByteStream中为读取做准备，不可信传输主要有以下几种情况：
1 传输有重叠，例如下标为1，长度为10的子串和下标为5，长度为3或者6的子串，分别对应全覆盖和部分重叠的情况。
2 传输顺序错误，例如下标为1的子串，晚于下标为6的子串到达重组器,定义为当前处理下标大于下一个处理下标。此时针对前端后端分别进行分类讨论即可，随时对当前处理下标进行更新。
除了不可信传输，传输容量大小也不能无限大，避免内存的滥用，实验限制如下所示：
<img width="416" alt="image" src="https://github.com/lawlietqq/CS144-TCP-IP/assets/92260319/6b0a3419-4c80-4b31-a878-917650d8ecdf">
上图从左到右分别为字节流读取后，未读取（已处理）ByteStream，未处理三段，重组器最大容量为绿色加红色的部分

流重组器的实现思路如下图所示：

<img width="412" alt="image" src="https://github.com/lawlietqq/CS144-TCP-IP/assets/92260319/a9693878-84ba-4313-90fa-a2d1f76f7344">

头文件新增私有成员如下：
std::map<size_t, std::string> _unassemble_strs;//下标与子串的映射
size_t _next_assembled_idx;//下一个待传入索引
size_t _unassembled_bytes_num;//未重排的所有字节数
size_t _eof_idx;//结束标志的索引
实验代码见sponge/libsponge/stream_reassembler.cc与sponge/libsponge/stream_reassembler.hh

Lab2 TCP接收器

实现tcp的接收部分，负责接收tcp段(实际的数据报有效负载)，重组字节流(包括它的结尾，当发生这种情况时)，并确定应该发送回发送方的信号以使发送方对已发送字节流进行确认并进行拥塞管理。

任务一：实现32位与64位的转换
在TCP中，SYN(流的开始)和FIN(流的结束)控制标志被分配了序列号。其中的每一个都占用一个序列号。(SYN标志占用的序列号是ISN)。流中的每个数据字节还占用一个序列号。SYN和FIN不是流本身的一部分，也不是“字节”，它们表示字节流本身的开始和结束。这些序列号(序列号)在每个TCP数据段的报头中传输，当前总共有三种索引，seqno是TCP流中的索引，而StreamReassembler中使用的索引对应stream index：

<img width="415" alt="image" src="https://github.com/lawlietqq/CS144-TCP-IP/assets/92260319/463bd837-11d8-471b-bb3b-bc035c4e6584">

1 序列号 seqno。从 ISN 起步，包含 SYN 和 FIN，32 位循环计数
2 绝对序列号 absolute seqno。从 0 起步，包含 SYN 和 FIN，64 位非循环计数
3 流索引 stream index。从 0 起步，排除 SYN 和 FIN，64 位非循环计数
2与3之间的转换比较简单，重点放在1与3之间的转换，需要实现64位与32位的互相转换，此处需要留意的是需要对偏移量与checkpoint的大小值进行比较并作不同处理

任务二：TCP Receiver
TCP receiver四种状态机的转化如下所示：

<img width="409" alt="image" src="https://github.com/lawlietqq/CS144-TCP-IP/assets/92260319/3d1ecdf0-7b6c-4226-bea3-4ae5b658fb8c">

对于 TCP Receiver 来说，除了错误状态以外，它一共有3种状态，分别是：LISTEN：等待 SYN 包的到来，若在 SYN 包到来前就有其他数据到来，则必须丢弃。SYN_RECV：获取到了 SYN 包，此时可以正常的接收数据包。FIN_RECV：获取到了 FIN 包，此时务必终止 ByteStream 数据流的输入。可以通过以下方式快速判断TCPReceiver的状态：当 isn未初始化时，肯定是LISTEN状态，当 ByteStream停止输入，是FIN_RECV状态，其他情况下，是SYN_RECV状态。需要我们实现三个函数，segment received()，ackno()和window size()，分别实现了接收字节流，获取窗口开头值，获取窗口大小的三个功能。

具体代码及注释见sponge/libsponge/tcp_receiver.cc以及sponge/libsponge/tcp_receiver.hh

Lab3 TCP Sender

TCP Sender实现的功能主要有以下几点：

跟踪接收方的窗口(接收传入的TCPReceiverMessages及其ACKNO和窗口大小)

读取字节流，尽可能填充窗口，创建新的TCP数据段(如果需要，包括SYN和FIN标志)，然后发送它们直到窗口已满或字节流没有更多要发送的数据段。

跟踪已发送但尚未被接收方确认的数据

超时重传

如何检测超时并进行重传？

TCP Sender必须跟踪其未完成的数据段，直到它们占据的序列号得到完全确认。TCPSender的所有者将定期调用TCPSender的tick方法，以指示时间的流逝并确定最早发送的数据段是否在未确认的情况下处于超时状态，否则需要重新传输(再次发送)。
每隔几毫秒，TCPSender就会调用tick方法，告诉它自上次调用该方法以来经过了多少毫秒。在构造TCPSender时，会传递一个参数，告诉它重传超时(RTO)的“初始值”。RTO是发送方等待接收方返回确认报文的最大时间。当发送方在RTO内未收到确认报文时，将触发重传机制。为了适应不同网络环境，发送方需要动态调整RTO值，但“初始值”保持不变。起始代码将RTO的“初始值”保存在名为initial retransmission timeout的成员变量中。每次发送包含数据的数据段时(无论是第一次还是重传)，如果计时器没有运行，则启动计时器，使其在RTO毫秒后超时(对于RTO的当前值)，当所有未完成的数据被确认后便停止重传计时器的计时。

如果调用了Tick且重传计时器已超时，需要进行如下工作：

(A)重新发送尚未被TCP接收器完全确认的最早(最低序列号)段

(B)如果接收者的窗口大小非零：

i.跟踪连续重传的次数，并进行线性增加，TCPConnection将使用此信息来确定连接是否连续重传太多次以及是否需要中止。

ii.使RTO的大小翻倍，这会减慢网络上的重传速度，避免进一步堵塞网络。

(C)重置并启动重传计时器，使其在RTO毫秒后超时(需要考虑RTO的值可能刚刚加倍)。

当接收者给发送者一个确认成功接收新数据的 ack 包时（absolute ack seqno 比之前接收到的 ackno 更大）：
将 RTO 设置回初始值

如果发送方存在尚未完成的数据，则重新启动重传定时器

将连续重传计数清零。

TCPSender的实现

1. void push( Reader& outbound stream );

TCPSender 从 ByteStream 中读取数据，并以 TCPSegement 的形式发送，尽可能地填充接收者的窗口。但每个TCP段的大小不得超过 TCPConfig::MAX PAYLOAD SIZE。

如果接收方已宣布窗口大小为0，则push方法应假装窗口大小为1。发送方可能最终发送一个被接收方拒绝(且未确认)的单个字节，但这也可能促使接收方发送一个确认段来告诉发送方最新窗口大小，不然发送者可能永远不会知道何时被允许再次开始发送。若远程没有 ack 这个在 window size 为 0 的情况下发送的一字节数据包，那么发送者重传时不要将 RTO 乘2。这是因为将 RTO 双倍的目的是为了避免网络拥堵，但此时的数据包丢弃并不是因为网络拥堵的问题，而是远程接收窗口过小。

2 void tick( const size t ms since last tick ):

自上次调用此方法以来经历的毫秒数。发送方可能需要重新传输未完成的数据段。

3 void send empty message(): 

生成并发送一个在 seq 空间中长度为 0 并正确设置 seqno 的 TCPSegment，允许用户发送一个空的 ACK 段。

4 void ack_received:

对接收方返回的ackno和window size进行处理。丢弃那些已经完全确认但仍然处于未确认队列的数据包,如果 window size 仍然存在空闲则继续发包。

具体代码及注释见sponge/libsponge/tcp_sender.cc以及sponge/libsponge/tcp_sender.hh

Lab4 TCPConnection

难度最大的一个lab.....状态机相当复杂，具体状态机以及最终实现TCP结构如下所示：

![image](https://github.com/lawlietqq/CS144-TCP-IP/assets/92260319/6af0e7ff-068f-4030-96c9-b082ab62cf0d)


![image](https://github.com/lawlietqq/CS144-TCP-IP/assets/92260319/b1e52e1d-12f1-4e09-842d-919af54e5736)

首先来看TCP的11种状态：
CLOSED：初始时没有任何连接的状态。

LISTEN：服务器监听来自客户端的连接请求(SYN包)。

SYN_SENT：客户端socket执行CONNECT连接，发送SYN包，之后等待来自服务器的SYN ACK包(服务器的连接请求和对客户端连接请求的确认)。

SYN_RCVD：服务端收到客户端的SYN包并发送服务端SYN ACK包，之后等待客户端对连接请求的确认(ACK包)。

ESTABLISH：表示连接建立。客户端发送了最后一个ACK包后进入此状态，服务端接收到ACK包后进入此状态。

FIN_WAIT_1：终止连接的一方（通常是客户机）发送了FIN包后进入此状态，之后等待对方FIN包。

CLOSE_WAIT：（假设服务器）接收到客户机FIN包之后等待关闭的阶段。在接收到对方的FIN包之后，自然是需要立即回复ACK包的，表示已经知道断开请求。但是本方是否立即断开连接（发送FIN包）取决于是否还有数据需要发送给客户端，若还有数据要发送，则在发送FIN包之前均为此状态。

FIN_WAIT_2：客户端接收到服务器的ACK包，但并没有立即接收到服务端的FIN包，进入FIN_WAIT_2状态。此时是半连接状态，即有一方要求关闭连接，等待另一方关闭。

LAST_ACK：服务端发动最后的FIN包，等待最后的客户端ACK包。

CLOSING：当主动关闭方处于FIN_WAIT_1时，被动关闭方的 FIN 先于之前的自己发送的 ACK 到达，主动关闭方就直接FIN_WAIT_1 -> CLOSING，（其实就相当于同时关闭)，然后迟来的 ACK 到达时，主动关闭方就从CLOSING -> TIME_WAIT。

TIME_WAIT：客户端收到服务端的FIN包，并立即发出ACK包做最后的确认，在此之后的2MSL(两倍的最长报文段寿命)时间称为TIME_WAIT状态。

TCP可靠地传输受流量控制的字节流，每个方向一个，两方参与TCP连接，每一方同时充当发送方和接收方，具体如下图所示：

![image](https://github.com/lawlietqq/CS144-TCP-IP/assets/92260319/8aaecc64-e25a-4e1b-bb0c-f4c922fe718b)

上图中的“A”和“B”称为连接的“端点”，TCPConnection负责接收和发送数据段，确保发送者和接收者被告知传入和传出数据段的字段。以下是TCPConnection必须遵循的基本规则：

Receiving segments.
如果设置了rst(重置)标志，则将入流和出流都设置为错误状态，并永久终止连接

将数据段交给TCPReceiver，以便它可以检查输入数据中需要关心的字段：Seqno、SYN、PayLoad和FIN

如果设置了ACK标志，则告知TCPSender它关心的输入数据段的字段：ACKNO和接收方窗口大小

如果传入数据段占用至少一个序列号，则TCPConnection确保至少发送一个数据段作为回复，以反映ackno和窗口大小的更新。

额外的特殊情况是响应“保持活动”的段。对等方可能会选择发送带有无效序列号的数据段，以查看您的TCPConnection是否仍然有效(如果是，则当前窗口大小是什么)，TCPConnection需要回复这些，即使它们不占用任何序列号。

Sending segments.

任何时候TCPSender将数据段推入其队列，并在传出数据段上设置它负责的字段：(seqno、syn、payload和fin)。在发送数据段之前，TCPConnection将向TCPReceiver询问其负责的传出数据段的字段：ackno和窗口大小。如果存在ACKNO，TCPConnection将设置ACK标志和TCPSegment中的字段。

TCPConnection具有操作系统定期调用的tick方法，告诉TCPSender已经经过的时间。如果连续重传的次数超过上限TCPConfig：：Max RETX尝试次数，则中止连接，并向对等方发送重置数据段(设置了rst标志的空数据段)。

TCP 连接的关闭是一个麻烦且重要的问题，主要有以下几种情况需要考虑：

在unclean shutdown中，TCPConnection发送或接收设置了rst标志的数据段。在这种情况下，出站字节流和入站字节流都应该处于错误状态，立刻执行退出。

在clean shutdown中，此时需要满足喜闻乐见的四次挥手的流程，具体流程如下所示：

  ![image](https://github.com/lawlietqq/CS144-TCP-IP/assets/92260319/b9ce7587-b9f3-46ea-a5fc-cfae54f20536)
  
在断开连接之前客户端和服务器都处于ESTABLISHED状态，双方都可以主动断开连接，以客户端主动断开连接为优。

第一次挥手：客户端打算断开连接，向服务器发送FIN报文(FIN标记位被设置为1，1表示为FIN，0表示不是)，FIN报文中会指定一个序列号，之后客户端进入FIN_WAIT_1状态。

也就是客户端发出连接释放报文段(FIN报文)，指定序列号seq = u，主动关闭TCP连接，等待服务器的确认。

第二次挥手：服务器收到连接释放报文段(FIN报文)后，就向客户端发送ACK应答报文，以客户端的FIN报文的序列号 seq+1 作为ACK应答报文段的确认序列号ack = seq+1 = u + 1。

接着服务器进入CLOSE_WAIT(等待关闭)状态，此时的TCP处于半关闭状态(下面会说什么是半关闭状态)，客户端到服务器的连接释放。客户端收到来自服务器的ACK应答报文段后，进入FIN_WAIT_2状态。

第三次握手：服务器也打算断开连接，向客户端发送连接释放(FIN)报文段，之后服务器进入LASK_ACK(最后确认)状态，等待客户端的确认。

服务器的连接释放(FIN)报文段的FIN=1，ACK=1，序列号seq=m，确认序列号ack=u+1。

第四次握手：客户端收到来自服务器的连接释放(FIN)报文段后，会向服务器发送一个ACK应答报文段，以连接释放(FIN)报文段的确认序号 ack 作为ACK应答报文段的序列号 seq，以连接释放(FIN)报文段的序列号 seq+1作为确认序号ack。

之后客户端进入TIME_WAIT(时间等待)状态，服务器收到ACK应答报文段后，服务器就进入CLOSE(关闭)状态，到此服务器的连接已经完成关闭。

客户端处于TIME_WAIT状态时，此时的TCP还未释放掉，需要等待2MSL后，客户端才进入CLOSE状态。

由客户端到服务器需要一个FIN和ACK，再由服务器到客户端需要一个FIN和ACK，因此通常被称为四次握手。

客户端和服务器都可以主动关闭连接，只有率先请求关闭的一方才会进入TIME_WAIT(时间等待状态)

具体实现代码见sponge/libsponge/tcp_connection.hh与sponge/libsponge/tcp_connection.cc

以上就是一个TCP的完整结构，lab5和lab6分别实现了IP地址与MAC地址之间的转换(ARP协议)，具有确认发送接口以及下一跳的IP 地址功能的路由器，因为以TCP为主体，不再详细叙述啦。
