# Goroutines和Channels

## Goroutines

- 类似其他语言的线程
- 同样没有执行顺序

## Channels

- 声明

        ch := make(chan int) //不带缓存
        ch := make(chan int, 0) //不带缓存
        ch := make(chan int, 100) //带缓存

- 写入写出

        ch <- x     //写入
        y := <-ch   //读出
        y, ok := <-ch //带错误类型的读取

- 遍历

        for v := range ch {
            fmt.Println(v)
        }

- 关闭

        close(ch)

### 不带缓存的channels

- 可以被称为同步Channels,作用类似c++中的竞争变量
- 可以用来阻塞主Goroutine,在子Goroutine执行完后使用channels来通知结束

### 串联的Channels(Pipeline)

- Channels 也可以用于将多个 goroutine 链接在一起,一个 Channels 的输出作为下一个Channels 的输入

- 使用

        func main() {
            naturals := make(chan int)
            squares := make(chan int)
            // Counter
            go func() {
                for x := 0; ; x++ {
                    naturals <- x
                }
            }()
            // Squarer
            go func() {
                for {
                    x := <-naturals
                    squares <- x * x
                }
            }()
            // Printer (in main goroutine)
            for {
                fmt.Println(<-squares)
            }
        }

### 单方向的Channel

- 当一个channel作为参数传递时,它总是被用于只发送或者只接收,所以channel可以指定具体的类型

        
        func counter(out chan<- int) {
            ...
        }

- 关闭操作只用于断言不再向channel发送新的数据,所以只有在发送的goroutine才会被调用close

- 不能将一个单向类型转换为双向类型

### 带缓存的Channels

- 可以理解为一个同步的消息队列
- 声明

        ch := make(chan int, 100) //带缓存

- 先进先出FIFO
- 在缓存没慢的情况下写入不阻塞,但是缓存满了还是会阻塞
- 使用cap来获取channel的容量

## select

- 语法和switch类似

        select {
        case <-ch1:
        // ...
        case x := <-ch2:
        // ...use x...
        case ch3 <- y:
        // ...
        default:
        // ...
        }

- 每个case后面需要是通讯操作
- 当条件满足的时候才会执行对应的case
- 没有时将阻塞
- 多个条件同时满足时随机选一个
- 有default时,没有满足的case情况下将执行default