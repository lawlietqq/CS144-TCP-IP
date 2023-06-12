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
在TCP中，SYN(流的开始)和FIN(流的结束)控制标志被分配了序列号。其中的每一个都占用一个序列号。(SYN标志占用的序列号是ISN。)。流中的每个数据字节还占用一个序列号。SYN和FIN不是流本身的一部分，也不是“字节”--它们表示字节流本身的开始和结束。这些序列号(序列号)在每个TCP数据段的报头中传输。(而且，同样，有两个流-每个方向一个。每个流都有单独的序列号和不同的随机ISN。)。“绝对序列号”(始终从零开始，不换行)和“流索引”(您已经在StreamReAssembly中使用的内容：流中每个字节的索引，从零开始)的概念同时共存。当前总共有三种索引：
1 序列号 seqno。从 ISN 起步，包含 SYN 和 FIN，32 位循环计数
2 绝对序列号 absolute seqno。从 0 起步，包含 SYN 和 FIN，64 位非循环计数
3 流索引 stream index。从 0 起步，排除 SYN 和 FIN，64 位非循环计数
2与3之间的转换比较简单，重点放在1与3之间的转换，需要实现64位与32位的互相转换
任务二：TCP receiver
TCP receiver四种状态机的转化如下所示：

