# GO基础学习笔记———函数

## 1.函数声明

- 函数声明包括:关键字(__func__),函数名,形参列表,返回值列表(可以省略),函数体

        func name (parameter-list) (result-list) {
            body
        }

- 和c++,python之类的不同,形参不支持默认赋值

- 传参格式

        func add(x, int, y int) int {return (x+y)}      //一般声明
        func sub(x, y int) (z int) {z = x - y; return}  //相同类型支持简写，return的时候可以指return时候的返回值名,然后直接返回
        func first(x int, _int) int {return x}          //＿代表不使用，占位
        func zero(int, int) int {return int}            　

- 形参的作用范围
    - 非指针的情况下是拷贝传递,修改形参不会影响原始值,引用类型下(指针,slice,map,function, channel等)会修改原始值

- 没有函数体的声明
    - 没有函数体的函数说明不是Go实现的,只是定义了函数标识符

            package math
            func Sin(x float64) float   //汇编实现

## 2.多返回值

- Go的返回和Python类似支持多返回值,标准库中多用于返回一个期望值和错误

        package main

        import (
            "fmt"
        )

        func main() {
            var ret, err = test(-5)
            //可以赋值也可以直接使用Println打印
            fmt.Println(ret, err)
            fmt.Println(test(-5))
        }

        func test(a int) (int, error) {
            if a > 0 {
                return a, nil
            } else {
                return a, fmt.Errorf("a<0")
            }
        }

- 如果不关心多返回的某个值,比如说错误则可以使用'_'代替

        var ret, _ = test(-5)
        fmt.Println(ret)

- 可以显示的声明返回值的变量名,这样return的时候可以直接返回,但是并不利于阅读,因为很容易忽略掉赋值的操作


        func test() (a int){
            a = 1
            return
        }

## 3.错误

其他语言错误返回一般是bool类型或者枚举类型,go使用内置的error类型可以返回具体的错误信息方便查找,error是一个接口类型,通过返回nil和non-nil来区分是否成功

### 3.1错误处理


### 3.2文件结尾错误(EOF)