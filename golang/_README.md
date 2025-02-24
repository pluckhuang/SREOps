### 1. 基本语法
1. defer, 作用, 执行顺序, 闭包变量引用、传参引用
2. 不要对迭代参数取地址
3. 切片内存共享问题
4. 可变参数, option 模式
5. 组合使用的的被组合的方法, 没有多态
6. 跨域请求:协议域名端口, 浏览器发送 preflight 请求询问, cors设置.

内存同步
```

Go 的内存模型通过以下几种操作来保证内存同步：

初始化：全局变量的初始化在程序开始执行前完成。
goroutine 创建：当创建一个新的 goroutine 时，父 goroutine 的所有对共享变量的写入在新 goroutine 开始前对其可见。
通道操作：
发送操作（channel <- value）在发送完成后，对发送值的写入对接收方可见。
接收操作（value := <- channel）保证在接收完成后，发送方对共享变量的写入对接收方可见。
锁操作（sync.Mutex 或 sync.RWMutex）：
Lock 确保在调用 Lock 前的任何写操作，对随后在 Unlock 后读取的 goroutine 可见。
Unlock 确保在 Unlock 前的任何写操作对随后获取锁的 goroutine 可见。
原子操作（sync/atomic 包）：
对同一个变量的原子操作有顺序保证。原子操作之间的内存效果是可见的。

内存顺序
happens before（发生在前）：如果一个事件 A "happens before" 事件 B，那么 A 的内存效果对 B 是可见的。Go 定义了一些 "happens before" 关系：
程序顺序：在单个 goroutine 内的操作按照源代码的顺序执行。
通道通信：发送操作 happens before 相应的接收操作完成。
锁：对同一个锁的解锁操作 happens before 后续的加锁操作。
原子操作：同一个变量的原子操作之间有顺序保证。

无序性
Go 没有保证未相关的操作的执行顺序，除非通过上述机制明确定义了顺序。在没有显式同步的情况下，两个 goroutine 可能以任意顺序执行其操作，这可能导致数据竞争。

数据竞争
如果两个 goroutine 并发访问同一个变量，并且至少有一个是写操作，而没有使用任何同步机制（如锁或通道），则存在数据竞争。Go 的竞争检测器（使用 go test -race 或 go run -race）可以帮助发现这些问题。
```

熔断器 [gobreaker](https://github.com/sony/gobreaker?tab=readme-ov-file) 使用研究说明:
```
熔断器默认为 close 状态, 阈值条件满足 ReadyToTrip 时, 熔断器进入 open 状态,  此后执行全部直接返回 false;
同时启动定时器 Timeout, 等到时间结束.熔断器进入 half-open 状态. 此时允许进行 MaxRequests 次执行, 以进行试探. 假如执行成功, 熔断器进入 close 状态, 变量重置.
否则, 进入open 状态, 同时再次开启定时器.
```

ESR 和 ISR 是什么
```
ESR 是 equal sort range 原则,  一般数据库索引的建立,可以按这个员额来建立
ISR 是 in sync replica, 代表 kafka 里面有效跟踪主节点的副本节点
```