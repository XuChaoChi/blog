# 如何优雅的结束go程序


### 使用WaitGroup

使用WaitGroup的好处：协程处理的任务的时候主线程阻塞，当所以协程结束的时候（即WaitGroup的级数为0的时候）停止阻塞执行剩下的任务

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func doClear() {
    fmt.Println("=====end=====")
}

func main() {
    wg := sync.WaitGroup{}
    wg.Add(1)
    go func() {
        //模拟任务
        for true {          
            time.Sleep(time.Second * 1)
            fmt.Println("di da...")
        }
        wg.Done()
    }()
    fmt.Println("=====Wait stop========")
    wg.Wait()
    doClear()
}
```

但是存在的问题在于如果是通过外部的信号停止的进程还是不能执行到doClear的逻辑，因为结束的时候主线程还是在阻塞状态，比如运行上面的代码，在终端按ctrl+c后并不能执行wg.Wait()后面的doClear()

```
输出
=====Wait stop========
di da...
di da...
di da...
```

### 使用信号

```go
package main

import (
    "fmt"
    "os"
    "os/signal"
    "syscall"
)

// 监听指定信号
func main() {
    //合建chan
    c := make(chan os.Signal)
    //监听指定信号 ctrl+c kill
    signal.Notify(c, os.Interrupt, os.Kill, syscall.SIGUSR1, syscall.SIGUSR2)
    //阻塞直到有信号传入
    fmt.Println("启动")
    //阻塞直至有信号传入
    s := <-c
    fmt.Println("退出信号", s)
}

```

### 使用go-svc

#### 简介

对linux下signal和windows下消息循环的统一封装，实现跨平台下的服务安全退出

#### 安装

```shell
go get -u github.com/judwhite/go-svc/svc
```

#### 官方例子

```go
package main

import (
	"log"
	"sync"

	"github.com/judwhite/go-svc/svc"
)

// program implements svc.Service
type program struct {
	wg   sync.WaitGroup
	quit chan struct{}
}

func main() {
	prg := &program{}

	// Call svc.Run to start your program/service.
	if err := svc.Run(prg); err != nil {
		log.Fatal(err)
	}
}

func (p *program) Init(env svc.Environment) error {
	log.Printf("is win service? %v\n", env.IsWindowsService())
	return nil
}

func (p *program) Start() error {
	// The Start method must not block, or Windows may assume your service failed
	// to start. Launch a Goroutine here to do something interesting/blocking.

	p.quit = make(chan struct{})

	p.wg.Add(1)
	go func() {
		log.Println("Starting...")
		<-p.quit
		log.Println("Quit signal received...")
		p.wg.Done()
	}()

	return nil
}

func (p *program) Stop() error {
	// The Stop method is invoked by stopping the Windows service, or by pressing Ctrl+C on the console.
	// This method may block, but it's a good idea to finish quickly or your process may be killed by
	// Windows during a shutdown/reboot. As a general rule you shouldn't rely on graceful shutdown.

	log.Println("Stopping...")
	close(p.quit)
	p.wg.Wait()
	log.Println("Stopped.")
	return nil
}
```

#### 源码分析

有2个统一接口，分别在linux下和windows下有不同的实现

```go
//服务的运行接口
type Service interface {
    //初始化
	Init(Environment) error
    //开始具体的业务逻辑
    Start() error
    //收到结束信号后调用
	Stop() error
}

//判断运行环境的接口
type Environment interface {
	IsWindowsService() bool
}
```