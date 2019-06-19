GO基础学习笔记———复合数据类型

## 1.数组

和c++一样数组不支持扩容，所以一般用slice替代会更灵活点

- 数组的长度用len函数获取
- 数组的初始化
    - 默认初始化的值是0
    - 可以和c++一样在开始的时候指定初始值

            var q[3]int = [3]int{1, 2, 3}
            var r[3]int = [3]int{1, 2}

    - go 也支持根据初始化的值来确定，不同于c++，go需要在括号里添加...

            q := [...]int{1, 2, 3}
            fmt.Printf("%T\n", q)//[3]int

    - 指定位置的初始化

            r := [...]int{99: -1}
            //表示定义了长度为100的数组并且最后一个元素的值被初始化位-1 

- 不同长度的数组是不同的类型，数组的长度必须是常量
- 数组的比较
  - 如果数组的类型可以相互转换则可以比较，可以使用==和!=来判断是否相同
  
        a := [2]int{1, 2}
        b := [...]int{1, 2}
        c := [3]int{1,2,3}
        fmt.Println(a == b, a == c) 
        // true false
        //不同长度的数组实际是不同的类型所以比较失败

## 2.Slice

Slice(切片)是Go中的变长数组，写作[]T,是元素的类型

- Slice的组成
    - 底层是引用了一个数组对象，由指针，长度(函数len返回)，容量组成(函数cap返回)

            type slice struct{
                array unsafe.Pointer
                len int
                cap int
            }

- 与python不同，不支持反向索引，实际范围是一个右半开的区间
- 多个slice可以引用同一个数组
- 当访问的索引超过长度并且小于容量的时候slice将扩容，超过容量的时候将抛出panic
- 数组和slice初始化的差异

        //数组
        a := [...]int{1, 2, 3}
        //slice
        a := []int{1, 2, 3}

- slice只支持与nil进行==比较
- 使用make创建，实际底层创建了一个匿名数组然后返回一个slice
    
        make([]T, len) 
        make([]T, len, cap) // same as make([]T, cap)[:len]

### 2.1 append函数

内置的append用于slice追加元素

### 2.2 slice内存技巧

- 一些函数传递中使用slice的时候可以引用相同的数组，这样可以减少内存的开辟

        package main

        import "fmt"

        func test(strings []string) []string {
                i := 0
                for _, v := range strings {
                        if v != "" {
                                strings[i] = v
                                i++
                        }
                }
                return strings[0:i]
        }

        func main() {
                strings := []string{"one", "", "three"}
                strings = test(strings)
                fmt.Println(rets)
                fmt.Println(strings)
        }
        //[one three]
        //[one three three]

    - 不过原来的数组会被覆盖
    - 使用一般以 strings = test(strings) 这样的方式

## 3.map

- map是kv的数据结构，本质是hashtable的引用
- key的比较必须通过==
- map的创建
    
        ages := make(map[string]int)
        ages := map[string]int{
            "xxx": 30,
            "bbb": 25
        }

- map元素的删除,操作是安全的即使没有这个key，如果map是nil则抛出panic

        delete(ages, "xxx")     //empty
        map["xxx"] = 10         //nil panic: assignment to entry in nil map


- map元素访问不存在的时候将返回0
- map中的元素并不是变量所以不能取地址，因为元素的增加map可能重新分配地址
- go中map和c++中最重要的一点不同是他是 __无序__ 的
  - 如果需要排序可以使用sort排序

        package main

        import "fmt"
        import "sort"

        func main() {
            ages := make(map[string] int)
            ages["bbb"] = 10
            ages["ccc"] = 22
            ages["aaa"] = 5
            names := make([]string, 0, len(ages))
            for name, _ := range ages{
                names = append(names, name)
            }
            sort.Strings(names)
            for _, name := range names {
                fmt.Println(name, ages[name])
            }
        }

- map插入成功后将返回成功的值，不成功返回0([string]int型)，为了判断插入的0值和是否成功，可以进行一下操作

        if age, ok := ages["bob"]; !ok { /* ... */ }

## 4.结构体

- 定义

        type Student struct{
            ID int
            Name string
        }

- 声明

        var sdt Student

- 访问使用

        sdt.ID = 10

- 通过指针使用成员

        id = &sdt.ID
        fmt.Println(*id)

- 结构体S中不能再包含S类型,但是可以包含*S类型的指针

## 5. JSON

- go结构体slice转json,只有导出的结构体成员才会被编码所以成员需要大写

        package main

        import (
        "fmt"
        "encoding/json"
        )

        type JsonSt struct {
        First string
        Second []int
        }

        func main() {
        var toJson = []JsonSt{
                {First:"first1", Second:[]int{1, 2, 3}},
                {First:"first2", Second:[]int{4, 5, 6, 7}},
        }
        //data, err := json.MarshalIndent(movies, "", "    ") //使用MarshalIndent返回缩进格式
        data, err := json.Marshal(toJson)
        if err != nil {
                fmt.Printf("parse err %s", err)
        }
        fmt.Printf("%s\n", data)
        }
        //结果[{"First":"first1","Second":[1,2,3]},{"First":"first2","Second":[4,5,6,7]}]

- json转go结构体slice

