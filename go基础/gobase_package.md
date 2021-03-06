# GO基础学习笔记———包

## 简介

- go标准库有100多个包，可以使用 __go list std | wc -l__ 来查看
- 包名类似c++的namespace可以防止冲突
- 可以通过大小写来实现封装，大写外部可见，小写不可见
- 包的导入必须再每个文件显式导入
- 禁止包的循环依赖

## 包声明

- 每个go语言源文件必须有包的声明语句
- 包名作为导入时候的默认标识符
- 默认包名是包路径的最后一段

## 导入声明

- 导入形式

        //第一种
        import "fmt"
        import "os"

        //第二种
        import (
            "fmt"
            "os"
        )

- 包导入可以使用空行来区分不同组织的包，如：

        import (
            "fmt"
            "heml/template"
        )

        "golang.org/x/new/heml"
        "golang.org/x/net/ipv4"

- 2个包名冲突的情况必须将一个其中的包取个别名

        import (
            "fmt"
            xfmt "xxx/fmt"
        )

## 包的匿名导入

- 为了处理一些只需要计算包级变量初始化表达式和执行导入函数的init初始函数
- 导入方式

        import _"image/png"

## 工具

### 包下载

- go get可以下载单独的包或者整个目录
- go get -u可以确保每次下载的都是最新的

### 包文档

- go doc可以查看整个包的文档
- go doc也可以精确到具体的成员或者方法的注释
- go doc也不需要区分大小写和具体的路径

## 内部包

- 有时候包的部分不想暴露，比如还不没写好之类的可以建一个internal目录,这样只能在同一个包访问

## 包查询

- go list 一个目录可以查询这个目录下的所有包
- go list -json 加上-json可以看到完整包的元信息
- 命令行参数-f 则允许用户使用 text/template 包的模板语言定义输出文本的格式