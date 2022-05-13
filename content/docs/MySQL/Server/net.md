MySQL packet的结构如下：

{{< hint info >}}
packet length (3 bytes)  	         			// 数据

packet number (1 byte)            			// 保序


compression length (3 bytes) optional    ``// 压缩
``

compression packet number （1 byte） optional // 保序

packet data
{{</hint>}}

因为采用3个字节存储包的长度，所以支持的包最大为 MAX_PACKET_LENGTH (256L*256L*256L-1)。如果数据流超过包最大值（16M），则通过packet number（ne→pkt_nr）保序。

# 发包

## my_net_write

发包，Write a logical packet with packet header.

## net_write_buff

发送缓冲区，Caching the data in a local buffer before sending it.

每个Net对象有一个buffer(net->buff)，即将发送的数据被拷贝到这个buffer中。如果buffer未满则进行memcpy，并更新写入点位（net->write_pos）；满了当Buffer满时需要立刻发出到客户端（net_write_packet）。

## net_write_packet

socket写数据，Write a MySQL protocol packet to the network handler.

## net_write_raw_loop

Write a determined number of bytes to a network handler.
调用vio_write进行socket写

## net_flush

Flush write_buffer if not empty.
也是调用net_write_packet进行socket写数据

# 收包

## my_net_read

收包

## net_read_packet

读单包，Read one (variable-length) MySQL protocol packet. A MySQL packet consists of a header and a payload.

## net_read_packet_header

读包头，Read the header of a packet.
校验包头中的packet number，确认保序
