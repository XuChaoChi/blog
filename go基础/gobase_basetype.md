---
title: GO基础学习笔记———基础数据类型
date: 2019-06-17 16:31:57
tags: go
category: go
keywords: go
---


# GO基础学习笔记———基础数据类型

## 1.数值型

### 1.1整形

- 固定长度的类型有：int8,int16,int32,int64和对应的无符号的uint8, uint16, uint32, uint64
- 根据CPU字节大小的有：int和uint,对于32位和64位平台分别占32和64位
- rune类型和int32等价，byte类型和uint8等价
- uintptr 足以存储指针的uint
  
<!--more-->

### 1.2浮点型

- float32和float64
- 浮点数的区间
  - 在math包中定义，如：math.MaxFloat32表示32位浮点的最大值
- 优先使用float64，因为float32的有效位只有23

        var f float32 = 16777216    // 1<<24
        fmt.Println(f == f+1)       //true
        var a float32 = 1.1234567899
        var b float32 = 1.12345678
        var c float32 = 1.123456781
        fmt.Println(a, b, c)            // 1.1234568 1.1234568 1.1234568
        fmt.Println(a == b, a == c)     // true true

- 除0的情况

        var z float64
        fmt.Println(1/z, -1/z, z/z)
        //结果：+Inf -Inf NaN
        //分别是正无穷，负无穷，无穷
    - 可以用math.IsNaN测试是否是NaN
    - NaN和任何数都不相等

### 1.3运算

- 科学型表示
    - 可以直接在后面添加e
     
            package main
            
            import (
                "fmt"
            )
            
            func main() {
                fmt.Println(int64(99.9e10))
                fmt.Println(int64(111e10))
            }
            //结果：
            //999000000000
            //1110000000000
- 运算符
    按照优先级
    |符号|作用|
    |-|-|
    |*|乘|
    |/|除|
    |%|取模|
    |<<|左移|
    |>>|右移|
    |&|AND|
    |&^|AND NOT|
    |+|加|
    |-|减|
    |\||OR|
    |^|XOR|
    |==|判断相等|
    |!=|判断不等|
    |<|小于|
    |<=|小于等于|
    |>|大于|
    |>=|大于等于|
    |&&|并|
    |\|\||或|

### 2.复数

一般用不上，用上了再说

### 3.布尔型

与c++不一样之处在于数值型不能隐式转换

### 4.字符串

#### 4.1 字符串面值

- 字符串的值不能被改变

        package main
        import (
                "fmt"
        )

        func main() {
                a := "test"
                a[1] = 'q'
                fmt.Println(a)
        }
        //./test.go:9:7: cannot assign to a[1]

- 子字符串操作:s[i:j]

        s := "hello, world" 
        fmt.Println(s[:5]) // "hello" 
        fmt.Println(s[7:]) // "world" 
        fmt.Println(s[:]) // "hello, world"

- 字符串连接

        s := "hello, world" 
        fmt.Println("goodbye" + s[5:]) // "goodbye, world"

- 字符串的对比
    - 可以直接使用 ==, >, <来对比，对比规则是从左到右逐个字节对比
  
#### 4.2 字符串编码

GO源文件采用UTF8编码

- Unicode 码点对应Go 语言中的rune 整数类型
- unicode/utf8 包则提供了用于rune 字符序列的UTF8 编码和解码
- 获取非纯英文(ASCII)的时候需要解码

        import (
            "fmt"
            "unicode/utf8"
        )

        func main() {
            s := "hello, 世界"
            fmt.Println(len(s))
            fmt.Println(utf8.RuneCountInString(s))
            fmt.Printf("%c\t\n", s[7])
            r, _ := utf8.DecodeRuneInString(s[7:])
            fmt.Printf("%c\t\n", r)
        }
        //13
        //9
        //ä
        //世

- range循环时将隐式解码

        s := "hello, 世界"
        for i, data := range s {
                fmt.Printf("%d\t%c\n", i, data)
        }
        //0       h
        //1       e
        //2       l
        //3       l
        //4       o
        //5       ,
        //6
        //7       世
        //10      界

- rune数组和string的互相转换
  - utf8->Unicode

        s := "hello, 世界"
        fmt.Printf("% x\n", s)
        r := []rune(s)
        fmt.Printf("% x\n", r)
        //68 65 6c 6c 6f 2c 20 e4 b8 96 e7 95 8c
        //[ 68  65  6c  6c  6f  2c  20  4e16  754c]

  - Unicode->utf8

        s := "hello, 世界"
        r := []rune(s)
        fmt.Printf(string(r))
        //hello, 世界

#### 4.3 字符串处理包

标准库中对字符串处理的包有bytes、strings、strconv和unicode包


#### 4.4 字符串和数字的转换

分别通过fmt.Printf和strconv包

- 数字转字符串

        a := strconv.Itoa(123)
        b := fmt.Sprintf("%d", 123)
        fmt.Println(a, b)
        //123 123

- 字符串转数字 

        x := 123
        //第二个参数表示转二进制
        a := strconv.FormatInt(int64(x), 2)
        //通过%b、%d、%o和%x转进制
        b := fmt.Sprintf("x=%b", x) 
        //x=1111011

- 字符串转整数

        x, err := strconv.Atoi("123") // 返回的是int
        y, err := strconv.ParseInt("123", 10, 64) // 后面2个参数表示10进制，64位，返回的永远是64位

### 5.常量

- 常量的运算在编译器完成,可以检测出如： 整数除零、字符串索引越界、任何导致无效浮点数的操作等等错误

- 常量运算的结果也是常量
- len、cap、real、imag、complex 和unsafe.Sizeof函数返回的也是常量
  
#### 5.1 常量声明
 
        const a int = 10    //一般声明
        const pi = 3.14149  //可以省略类型名
        const (             //批量声明
            e = 2.71828
            pi = 3.14159
        )

        const ( //第一个常量必须显示初始化，后面的没有指定初始化值的话和前一个值相同
            a = 1 
            b
            c = 2 
            d ) 
        fmt.Println(a, b, c, d) // "1 1 2 2"


#### 5.2 iota常量生成器

有点像c++中的enum，go中没有特定的枚举类型，一般用iota来定义

    type Flags uint//相当于枚举类型

    const (
            One Flags = iota//相当于第一个枚举值
            Two
            Three
    )

    func main() {
            fmt.Println(One, Two, Three)
    }
    //0 1 2

复杂情况的操作

    type Flags uint

    const (
            One Flags = 1 << iota   //相当于1 << 0
            Two                     //相当于1 << 1
            Three                   //相当于1 << 2
    )

    func main() {
            fmt.Println(One, Two, Three)
    }
    //1 2 4

#### 5.3 无类型的常量

- go中有很多常量没有明确的基础类型，但是这些类型至少有256位的运算精度，这样可以提高精度

- 六种常量类型：无类型的布尔型、无类型的整数、无类型的字符、无类型的浮点数、无类型的复数、无类型
的字符串

- 延迟加载可以不用显示转换，例子:

        var x float32 = math.Pi 
        var y float64 = math.Pi 
        var z complex128 = math.Pi
