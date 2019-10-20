---
title: nsq源码阅读笔记--日志模块(lg)
tags:
  - nsq
  - go
category: 'nsq'
keywords: 'nsq,go'
date: 2019-09-12 01:35:05
---

# nsq源码阅读笔记--日志模块(lg)

### 代码地址

[nsq-日志模块阅读](https://github.com/XuChaoChi/nsq_read/tree/master/internal/lg)


### 简介

nsq中将标准库中的log模块又进行了简单的封装，目录在nsq项目中的 __internal/lg__ 目录，支持日志等级

### lg模块结构

#### 日志等级

- 使用int类型来定义日志等级

    ```golang
    type LogLevel int      
    ```

- 日志等级分别如下，可以学习到定义同一类型常量前可以使用重定义来明确枚举的目的

    ```golang
    const (
        DEBUG = LogLevel(1)
        INFO  = LogLevel(2)
        WARN  = LogLevel(3)
        ERROR = LogLevel(4)
        FATAL = LogLevel(5)
    ) 
    ```
<!--more-->

- 日志等级相关的函数

    ```golang
    //根据字符串获取对应的日志等级（不分大小写）
    ParseLogLevel(levelstr string) : LogLevel, error

    ```

- 日志等级相关的成员方法

    值得注意的是Get()的返回interface{} 作用是方便类型转换可以直接和int进行比较

    ```golang
    //获取日志等级
    func (l *LogLevel) Get() : interface{}
    //设置日志等级
    func (l *LogLevel) (s string) : error
    //获取日志等级的对应字符串
    func (l *LogLevel) String() : string
    ```


#### Logger接口

- 接口定义
    
    ```golang
    type Logger interface {
        //写入函数
        Output(maxdepth int, s string) error
    }
    ```

- 空接口的实现

    在lg.go中有一个 __NilLogger__ 实现了Logger接口，搜索了下主要在测试中用到

    ```golang
    ./nsqadmin/nsqadmin_test.go:19:	opts.Logger = lg.NilLogger{}
    ./nsqadmin/nsqadmin_test.go:28:	opts.Logger = lg.NilLogger{}
    ```

#### 日志函数

- 日志输出调用
    
    ```golang
    //Param1 log对象
    //Param2 配置的级别
    //Param3 当前消息的等级
    func Logf(logger Logger, cfgLevel LogLevel, msgLevel LogLevel, f string, args ...interface{}) {
        //消息级别没有配置高直接返回
        if cfgLevel > msgLevel {
            return
        }
        logger.Output(3, fmt.Sprintf(msgLevel.String()+": "+f, args...))
    }
    ```

- 直接输出错误

    用于产生严重错误时，可能正常的日志对象还没有被构建，调用后将退出进程

    ```golang
    //在没有日志对象的情况下输出严:重错误日志
    func LogFatal(prefix string, f string, args ...interface{}) {
        logger := log.New(os.Stderr, prefix, log.Ldate|log.Ltime|log.Lmicroseconds)
        Logf(logger, FATAL, FATAL, f, args...)
        os.Exit(1)
    }
    ```


#### 日志函数重定义

用于函数中将日志函数作为参数传递

```golang
type AppLogFunc func(lvl LogLevel, f string, args ...interface{})
```

### lg的使用

```golang
package main

import (
    "log"
    "os"

    "github.com/nsqio/nsq/internal/lg"
)

const (
    LOG_DEBUG = lg.DEBUG
    LOG_INFO  = lg.INFO
    LOG_WARN  = lg.WARN
    LOG_ERROR = lg.ERROR
    LOG_FATAL = lg.FATAL
)

type TestLogger lg.Logger

var testLog TestLogger

func logf(level lg.LogLevel, f string, args ...interface{}) {
    lg.Logf(testLog, LOG_DEBUG, level, f, args...)
}

func main() {
    //因为标准库的包实现了Output所以直接使用就可以了
    testLog = log.New(os.Stderr, "[test]", log.Ldate|log.Ltime|log.Lmicroseconds)
    logf(LOG_DEBUG, "test:%d", 1)
    doAgain(logf)
}

func doAgain(lgFunc lg.AppLogFunc) {
    lgFunc(LOG_INFO, "test:%d", 2)
}


//输出结果:
//[test]2019/09/12 01:28:08.827837 DEBUG: test:1
//[test]2019/09/12 01:28:08.828021 INFO: test:2
```