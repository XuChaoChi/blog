---
title: GO基础学习笔记———方法
date: 2019-06-20 17:30:57
tags: go
category: go
keywords: go
---


# GO基础学习笔记———方法

## 1.方法的声明

- 方法的声明与函数不同之处在于在func后面要首先声明函数的接收器
  
        package main
        import (
            "fmt"
            "math"
        )

        type Point struct {
            X, Y float64
        }
        //普通函数
        func Distance (p, q Point) float64 {
            return math.Hypot(p.X - q.X, p.Y - q.Y)
        }
        //方法
        func (p Point) Distance (q Point) float64 {
            return math.Hypot(p.X - q.X, p.Y - q.Y)
        }

        func main() {
            p := Point{1, 10}
            q := Point{20, 10}
            fmt.Println(Distance(p, q))
            fmt.Println(p.Distance(q))
        }

- 在上面的方法中，p被称为方法的接收器
- 结构体中不能有和方法同名的字段

<!--more-->

## 2.基于指针对象的方法

- 如果参数的过大希望避免拷贝的时候就可以使用指针作为接收器，如下:(*Point).ScaleBy就是方法名

        func (p *Point) ScaleBy(factor float64) { 
            p.X *= factor 
            p.Y *= factor 
        }

- 为了避免歧义，一个类型名本身是指针的话，是不允许出现在接收器中的：

        type P *int
        func (P) f() P{} // compile error: invalid receiver type

- 调用方式

        //第一种
        r := &Point{1, 2}
        r.ScaleBy(2)
        //第二种
        p := Point{1, 2}
        ptr := &p
        ptr.ScaleBy(2)
        //第三种
        p := Point{1, 2}
        (&p).ScaleBy(2)
        //第四种（推荐）
        p.ScaleBy(2)

    - 编译器会隐式的&来调用指针方法，不过这种写法只是适用于变量

            Point{1, 2}.ScaleBy(2) //不能取到地址

- nil也是一个合法的接收器类型

## 3. 通过嵌入结构体来扩展类型

- 在作用上有点像其他面向对象的继承，内层的结构体被看作基类，但是实际上并不是继承


        package main
        import (
            "fmt"
            "math"
        )

        type Point struct {
            X, Y float64
        }

        type PointEx struct {
            Point
            ex int
        }

        func (p Point) Distance (q Point) float64 {
            return math.Hypot(p.X - q.X, p.Y - q.Y)
        }

        func main() {
            q := Point{20, 10}
            ex := PointEx{Point{1, 10}, 1}
            fmt.Println(ex.Distance(q))
        }

- 通过内嵌字段来调用，如上面的PointEx,可以直接使用Point的方法
    - 从实现角度，可以理解为编译器在原来的方法外面包了一层

            func (p PointEx) Distance(q Point) float64 { 
                return p.Point.Distance(q) 
            } 

    - 这种方式是递归的如果同一层有相同的方法则会报错

## 4.方法值和方法表达式

### 4.1 方法值

- 和c++中类的回调函数差不多
- p.Distance()中p.Distance叫做选择器，类似c++中的函数指针

        package main
        import (
            "fmt"
            "math"
        )

        type Point struct {
            X, Y float64
        }

        func (p Point) Distance (q Point) float64 {
            return math.Hypot(p.X - q.X, p.Y - q.Y)
        }

        func main() {
            p := Point{1, 2}
            q := Point{4, 6}
            disFunc := p.Distance
            fmt.Println(disFunc(q))
        }

### 4.2方法表达式

- 类似c++中的std::function和std::bind

        package main
        import (
            "fmt"
            "math"
        )

        type Point struct {
            X, Y float64
        }

        func (p Point) Distance (q Point) float64 {
            return math.Hypot(p.X - q.X, p.Y - q.Y)
        }

        func main() {
            p := Point{1, 2}
            q := Point{4, 6}
            disFunc := Point.Distance
            fmt.Println(disFunc(p, q))
        }

- 调用的时候的第一个参数需要选择接收器

## 5 封装

Go 语言只有一种控制可见性的手段：大写首字母的标识符会从定义它们的包中被导出，小写
字母的则不会。这种限制包内成员的方式同样适用于struct 或者一个类型的方法，其他好像和其他语言差不多