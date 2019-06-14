---
title: GO基础学习笔记———程序结构
date: 2019-06-14 12:17:43
tags: go
category: go
keywords: go
---


# GO基础学习笔记———程序结构

## 1.命名

### 1.1 关键字

- var 声明变量
- const 声明常量
- package 包名
- import 导入包
- func 声明函数
- return 用于从函数返回
- defer 推迟到return前执行函数
- go 用于并行
- select 用于选择不同类型的通讯
- interface 定义接口
- struct 定义结构体
- break、case、continue、for、fallthrough、else、if、switch、goto、default  流程控制
- chan 用于channel通讯，类似消息队列
- type 用于声明自定义类型，类似typedef
- map 用于声明map类型数据，但是无序
- range 用于读取slice、map、channel数据，和python的range类似

关键字不能用于命名，只能在特定的结构中使用

<!--more-->

### 1.2 内置类型

- 常量型：true, false, iota, nil 
- 值类型：int(32 or 64), int8, int16, int32, int64, uint(32 or 64), uint8(byte), uint16, uint32, uint64, float32, float64, string, complex64, complex128, array
- 函数型：make, ;en, cap, new, append, copy, close, delete, complex, real, imag, panic, recover

内置类型不是关键字，可以重新定义

    //测试
    package main
    import (
        "fmt"
    ) 
    func main() {
        var true bool
        fmt.Println(true)
        new()
    }
    func new() {
        fmt.Println("new")
    }
    ////////////////
    //结果：
    //false
    //new
    ////////////////

## 2.变量

### 2.1 变量的普通声明

- 普通声明的格式 __var 变量名 类型 = 表达式__
- 变量没有提供初始化值会初始化为2进制零值
  
        var x int       //自动初始化为零

- 显示提供初始化值，可以省略类型
  
        var y = false   //推断类型为bool

- 一次可以定义多个相同类型或者不同类型的变量
    
        var i, j, k int                     //都为int
        var b, f, s = true, 2.3, "four"     //bool, float, string

- 通过函数返回
    
        var f, err = os.Open(name)

### 2.2 变量的简短声明

- 简短声明的格式 __变量名 := 表达式
- 简短声明例子

        a := 10
        a, b := 1, false //多个变量

- 函数返回可以对已有的变量赋值，新的变量声明，但是最少要有一个新的变量

        var a = 10
        fmt.Println(&a)
        a, b := func() (int, bool) {
            return 1, false
        }()
        fmt.Println(&a, b)
        //结果
        //0xc420012098
        //0xc420012098 1 false

    可看a还是原来的地址，只是赋值

        var a, b = 10, false
        fmt.Println(&a)
        a, b := func() (int, bool) {
            return 1, false
        }()
        fmt.Println(&a, b)
        //错误提示: no new variables on left side of :=
    
    可见必须要有一个是新的变量

### 2.3 指针

与c/c++相比
- go的指针不能，进行运算
  - *p++ 是p指向的值++
- 指针不一定是在堆上分配，同样的，普通的变量也可能被分配在堆上面，取决于编译器，认定的变量的作用域

        //这个返回的指针是有效的
        func f() *int{
            v := 1 
            return &v
        }

- 普通取值

        var a = 10
        p := &a

- 使用new()
 
        p = new(int) //指向为零的匿名int

## 3.赋值

- 普通赋值
  - x = 1
- 二元算数符和赋值语句符合 
  - x += 1 
- ++,--赋值
  - 区别于从c/c++,没有前后自增之说必须放在变量后面,(--x)不符合
  - 不能作为右值被别的值赋值,(a = x++)不符合
- 元组赋值
  - 允许同更新多个值下，并且先计算右边的值，再赋值给左边，x, y = y, x / 2 

## 4.类型

- 重命名，类似c++中的typedef和using
  - type 类型名 底层类型
- 访问权限
   - 首字母大写才能被其他的包访问，中文默认是小写，所以使用中文type不能被其他包访问  
- 类型转换
  - 类型转换格式T(x),指针需要括号包裹(*T)(x)
  - 与c++不同大部分情况需要显示转换，即使底层类型相同也需要

        type TypeT int32  
        func main() {
            var a TypeT = 10
            var b int32 = 10
            if a == b {
                fmt.Println("ok")
            }
        }
        //invalid operation: a == b (mismatched types TypeT and int32)


## 5.包

- 包的声明
  - package xxx, 访问成员时可以用xxx.Member 
   
        // main.go
        package main
        import (
                "fmt"
        )
        func main() {
                fmt.Println(TValue)
        }

        //mainTest.go
        package main
        var TValue = 10
    运行 go run main.go mainTest.go 得到10
- 包的导入
 
        import "fmt"    //第一种方式
        import (        //第二种方式
            "fmt"       
        )
- 一个包只有一个源文件有包注释，如果过大可以放到单独的一个doc.go文件