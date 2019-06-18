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

- 错误处理方式一:检测错误是否是nil,不是的话直接返回错误信息或者再完善下错误信息的内容

        resp, err := http.Get(url)
        if err != nil {
            return nil, err
            //完善错误信息的内容更便于查找错误
            //return nil, fmt.Errorf("http Get err: %v", err)
        }

- 错误处理方式二:在错误发生是偶然性的,或者不可预知的情况,一是设置一个有时间间隔或者次数的重试机制

- 错误处理方式三:直接退出程序(os.Exit(1)),不过只应该在main中,其他位置的应该向上传播

- 错误处理方式四:通过log包输出错误日志

- 错误处理方式五:直接忽略掉......


### 3.2文件结尾错误(EOF)

io包保证任何由文件结束引起的读取失败都返回同一个io.EOF错误,错误的定义如下:

    import io
    import "error"
    var EOF = errors.New("EOF")

测试代码如下:

    in := bufio.NewReader(os.Stdin)
    for {
        r, _, err := in.ReadRune()
        if err == io.EOF {
            break // finished reading
        }
        if err != nil {
            return fmt.Errorf("read failed:%v", err)
        }
    }

## 4.函数值

- 在Go中函数和其他值一样拥有类型,可以被赋值和传递参数,以及返回,功能类似c++中的函数指针和仿函数
- 函数值为nil的时候进行访问会抛出
- 函数之间不能比较,并且不能作为map的key

## 5.匿名函数

匿名函数指的是没有定义名字符号的函数,可以作为变量,参数和返回值

### 5.1闭包函数

闭包是函数和其引用环境的组合体,例子:

    package main

    import (
        "fmt"
    )

    func test(x int) func() {
        fmt.Println(&x)
        return func() {
            fmt.Println(&x)
        }
    }
    func main() {
        a := test(10)
        a()
    }
    //结果:
    //0xc420012088
    //0xc420012088

可见x实际是上下文中的引用

### 5.2 迭代变量

正如闭包函数的特特性,容易造成迭代变量的问题(循环的时候闭包函数引用的同一个迭代值,最后处理的的是最后一次迭代的结果)

    package main

    import (
        "fmt"
    )

    func main() {
        var funcArr [5]func()
        funcSlice := funcArr[:]
        var strArr = [...](string){"one", "two", "three"}
        for _, v := range strArr {
            funcSlice = append(funcSlice, func() {
                fmt.Println(v)
            })
        }

        for _, v := range funcSlice {
            if v != nil {
                v()
            }
        }
    }

    //结果
    //three
    //three
    //three

处理方式,只要在迭代变量赋值的时候使用一个零时变量就可以了

    for _, v := range strArr {
		newv := v //在这里用临时变量赋值
		funcSlice = append(funcSlice, func() {
			fmt.Println(newv)
		})
	}

    ///结果
    //one
    //two
    //three

## 6.可变参数

- 声明:在参数类前加上 ...

- 例子如下:

        package main
        import (
            "fmt"
        )

        func sum(vals ...int) int {
            fmt.Printf("%T\n", vals)
            total := 0
            for _, val := range vals {
                total += val
            }
            return total
        }

        func main() {
            fmt.Println(sum(1, 2, 3))
            fmt.Println(sum([]int{1, 2, 3}...))
        }
        //结果
        //[]int
        //6
        //[]int
        //6

- 可见可变参函数的传递时被看做[]int的切片
- 如果原始参数本来就是切片,秩序传递的时候在变量后面添加...
- 虽然传递的时候被转换成切片但是本质和切片是不同的

        func f(...int) {}
        func g([]int) {}
        fmt.Printf("%T\n", f) // "func(...int)"
        fmt.Printf("%T\n", g) // "func([]int)"

## 7.Deferred函数

- defer 语句经常被用于处理成对的操作,如打开、关闭、连接、断开连接、加锁、释放锁
- 在一个函数里重复defer以FILO规则,可以理解是按入栈出栈规则处理
- 可以用defer来计算函数的耗时,如:

        package main
        import (
            "fmt"
            "time"
        )
        func test(msg string) func() {

            start := time.Now()
            fmt.Printf("enter %s\n", msg)
            return func() {
                fmt.Printf("exit %s,(%s)\n", msg, time.Since(start))
            }
        }

        func main() {
            defer test("test....")()
            time.Sleep(2 * time.Second)
        }
        //enter test....
        //exit test....,(2.000193984s)        
