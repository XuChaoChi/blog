# nsq日志模块学习

### 代码地址

[nsq-日志模块阅读](https://github.com/XuChaoChi/nsq_read/tree/master/internal/lg)


### 简介

nsq中将标准库中的log模块又进行了简单的封装，目录在nsq项目中的 __internal/lg__ 目录，支持日志等级

### lg模块结构

#### 日志等级

- 使用int类型来定义日志等级

        type LogLevel int      

- 日志等级分别如下，可以学习到定义同一类型常量前可以使用重定义来明确枚举的目的

        const (
            DEBUG = LogLevel(1)
            INFO  = LogLevel(2)
            WARN  = LogLevel(3)
            ERROR = LogLevel(4)
            FATAL = LogLevel(5)
        )   

- 日志等级相关的函数

        //根据字符串获取对应的日志等级（不分大小写）
        ParseLogLevel(levelstr string) : LogLevel, error


- 日志等级相关的成员方法

    值得注意的是Get()的返回interface{} 作用是方便类型转换可以直接和int进行比较

        //获取日志等级
        func (l *LogLevel) Get() : interface{}
        //设置日志等级
        func (l *LogLevel) (s string) : error
        //获取日志等级的对应字符串
        func (l *LogLevel) String() : string


#### Logger接口

- 接口定义
    
        type Logger interface {
            //写入函数
            Output(maxdepth int, s string) error
        }

- 空接口的实现

    在lg.go中有一个 __NilLogger__ 实现了Logger接口，搜索了下主要在测试中用到

        ./nsqadmin/nsqadmin_test.go:19:	opts.Logger = lg.NilLogger{}
        ./nsqadmin/nsqadmin_test.go:28:	opts.Logger = lg.NilLogger{}

#### 日志函数

- 日志输出调用
  
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

- 直接输出错误

    用于产生严重错误时，可能正常的日志对象还没有被构建，将退出进程

        //在没有日志对象的情况下输出严:重错误日志
        func LogFatal(prefix string, f string, args ...interface{}) {
            logger := log.New(os.Stderr, prefix, log.Ldate|log.Ltime|log.Lmicroseconds)
            Logf(logger, FATAL, FATAL, f, args...)
            os.Exit(1)
        }


#### 日志函数重定义

用于函数中将日志函数作为参数传递

    type AppLogFunc func(lvl LogLevel, f string, args ...interface{})


### lg的使用
