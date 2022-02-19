



protobuf 编码

https://taoshu.in/pb-encoding.html

https://mp.weixin.qq.com/s/VNy2A7WfRXstOcdakDhajw protobuf使用

https://mp.weixin.qq.com/s/2JkJ1iW_Vds1YaLlS5rudg protobuf编码

```
<tag> <type> [<length>] <data>

Protocol Buffers 在 3 版本中定义了 4 种类型 type：

0 VarInt 表示 int32, int64, uint32, uint64, sint32, sint64, bool, enum
1 64-bit 表示 fixed64, sfixed64, double
2 Length-delimited 表示 string, bytes, embedded messages, repeated 字段
5 32-bit 表示 fixed32, sfixed32, float  

7       0 7      0
 +-----+---+--------+
 |00001|000|00000001|
 +-----+---+--------+
   tag type   data

  7       0 7      0 25        0
 +-----+---+--------+===========+
 |00010|010|00000011|0xe50x90x95|
 +-----+---+--------+===========+
   tag type  length    utf-8
   
   
   
   Protocol Buffers 还有一个问题需要注意，那就是 tag 的取值范围。因为使用了 VarInts，所以单字节的最高位是零，而最低三位表示类型，所以只剩下 4 位可用了。也就是说，当你的字段数量超过 16 时，就需要用两个以上的字节表示了。
   
   
不要修改字段 tag
字段尽量不要超过 16 个
尽量使用小整数
如果需要传输负数，请使用 sint32 或 sint64
```



type

| Type | Meaning          | Used For                                                 |
| :--- | :--------------- | :------------------------------------------------------- |
| 0    | Varint           | int32, int64, uint32, uint64, sint32, sint64, bool, enum |
| 1    | 64-bit           | fixed64, sfixed64, double                                |
| 2    | Length-delimited | string, bytes, embedded messages, packed repeated fields |
| 3    | Start group      | groups (deprecated)                                      |
| 4    | End group        | groups (deprecated)                                      |
| 5    | 32-bit           | fixed32, sfixed32, float                                 |

流式消息的每个`key`的值格式均为`(field_number << 3) | wire_type`,也就是说，数字的后三位存储了`wire type`。



Varint

https://blog.csdn.net/weixin_40021744/article/details/86638612
