### ICMP



[Internet Control Message Protocol - Wikipedia](https://en.wikipedia.org/wiki/Internet_Control_Message_Protocol#Control_messages)

https://www.huaweicloud.com/articles/d83085564a03ef25d11f0d38f8982213.html

https://www.rt-thread.org/document/site/tutorial/qemu-network/ping_principle/ping_principle.pdf

[使用 golang 实现 ping 命令 | 始于珞尘 (juejin.cn)](https://juejin.cn/post/6844903574833479688)

[JustDoIt/goping.go at master · hiberabyss/JustDoIt (github.com)](https://github.com/hiberabyss/JustDoIt/blob/master/ping/goping.go)

一个ICMP报文包括**IP报头（至少20字节**）、**ICMP报头（至少八字节）**和ICMP报文（属于ICMP报文的数据部分）



```go
type ICMP struct {
	Type        uint8       // 类型     
	Code        uint8				// 进一步划分类型        ping是echo type=8 code=0
	CheckSum    uint16			// 报文头的校验值  先设置0 算完再赋值
	Identifier  uint16 			// 
	SequenceNum uint16			// 序列号 递增
}
```

