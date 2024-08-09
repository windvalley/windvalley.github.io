---
title: "Go编程的小Tips"
date: 2017-11-15T10:38:52+08:00
authors:
  - windvalley
categories:
  - Go
tags:
  - go
  - tips
draft: true
---

## 常识

Go 的正确叫法和写法是`Go`, 而不是`golang` or `GO` or `go` or others.

```txt
Is the language called Go or Golang?

The language is called Go.
The "golang" moniker arose because the web site is golang.org, not go.org,
which was not available to us.
Many use the golang name, though, and it is handy as a label.
For instance, the Twitter tag for the language is "#golang".
The language's name is just plain Go, regardless.
A side note: Although the official logo has two capital letters,
the language name is written Go, not GO.

https://golang.org/doc/faq
```

## Go 相比 Python 的优劣势

优势:

- `Go`为并发而生, 因为`goroutine`以及`channel`,
  天然支持高并发(并行), 可利用多核 CPU.
- `Go`开发出的程序性能远高于`Python`,
  一个 Go 进程可能比 200 个 Python 进程的处理能力更强, 而且 CPU 和内存占用更低.
- Go 是静态语言、强类型, 静态编译能帮我们提前检查出来大量的错误.
- 运维和部署简单, 只需要一个二进制文件,
  因为所有的依赖都编译到了这个二进制文件中.
- `gofmt`使 Go 代码整体风格一致.
- 适合做大项目, 对于团队成员代码能力良莠不齐的情况, `Go`是更好的选择.

劣势:

- 编码效率大概是`Python`的 85%.
- 因为`Go`相对年轻很多, 所以生态相对`Python`差了很多.

## go run

为了代码结构清晰, 当把一个包代码文件拆分成**同目录**下的多个文件的时候(同属一个 package),
需要注意的事项:

- go run main.go 执行的时候, 要把其他的分拆文件追加到后面, 比如:
  go run main.go xxx.go yyy.go(go 文件的顺序无关), 否则会报错.
  或者`go run *.go`这样执行.

- `go build`对项目进行编译, 执行不会报错.

- 分拆的代码文件中的各种变量名可以不用是首字母大写即可互相引用.

## os.Chdir()

如果我们写的应用程序依赖外部配置文件或会记录产生的日志,
为了让编译后的二进制文件能以绝对路径正确执行, 比如/path/cmd, 我们在 init 函数中
切换路径到 cmd 的所在路径下即可.

```go
func init() {
    // 切换到当前二进制命令所在目录
    basedir := filepath.Dir(os.Args[0])
    _ = os.Chdir(basedir)
}
```

注意: 如果是`go run *.go`测试, 需要将这段临时注释, 因为 go run 会在/tmp 目录
创建临时的目录进行编译, 这段代码会将目录切换到这个临时目录中, 出现找不到依赖的
配置文件的错误.

## 简易记录日志

记录日志到本地文件的简单用法:

```go
var (
	logdir = "./log/"
	logger *log.Logger
)

func init() {
	// 切换到当前二进制命令所在目录
	basedir := filepath.Dir(os.Args[0])
	_ = os.Chdir(basedir)

	// log目录不存在则创建
	_, err := os.Lstat(logdir)
	if os.IsNotExist(err) {
		os.Mkdir(logdir, 0755)
	}

    // 定义日志文件名称
	fileName := "error-" + time.Now().Format("20060102") + ".log"
	logFile, err2 := os.OpenFile(logdir+fileName,
		os.O_RDWR|os.O_CREATE|os.O_APPEND, 0644)
	if err2 != nil {
		panic(err2)
	}

    // 获取记录日志的对象
	logger = log.New(logFile, "", log.LstdFlags|log.Lshortfile|log.Ltime)

	return
}
```

调用方法举例:
`logger.Printf("Get() err: %v, %s,%s", err2, i.IpAddr, i.index)`

## Go 类型的零值

当变量等被创建时如果没有明确指定初始值, Go 语言会自动初始化其值为此类型对应的零值.

各类型的零值:

- bool: `false`
- integer: `0`
- float: `0.0`
- string: `""`
- pointer/function/interface/slice/channel/map: `nil`
- array/struct: `递归的将每一个元素初始化为其类型对应的零值`

## 复合数据类型判断

使用标准库或第三方包时,
如果想判断其中的某个自定义复合数据类型(比如结构体等)是否有效时,
看看该复合数据类型的定义中是否有可以直接比较的字段类型, 例如
`int`,`string`,`slice`等简单类型, 比如若存在`slice`类型(`[]type`),
则使用`slice == nil` 或`slice != nil`来判断 slice 是否为空.

## go vet

`go vet`
对当前目录下的 Go 源码做静态检查, 以发现潜在的问题.

## time 库相关

`time.Now().Unix()`, 返回当前的时间戳.

时间向下取整分钟, 返回 time.Time 类型: `time.Now().Truncate(time.Minute)`

时间四舍五入取整小时, 返回 time.Time 类型: `time.Now().Round(time.Hour)`

将时间字符串`"2016-11-22 19:00:00"`转化为 time.Time 类型, 并且使用本地时区:
`time.ParseInLocation("2006-01-02 15:04:05", "2016-11-22 19:00:00", time.Local)`

返回 10 分钟之前的格式化时间, 返回 string 类型:
`time.Now().Add(-time.Minute * 10).Format("2006-01-02 15:04:05")`

和上面不同的是, 这个返回 UTC 时间, 而不是本地时间:
`time.Now().UTC().Add(-time.Minute * 10).Format("2006-01-02 15:04:05")`

格式化时间的另一种方法:

```go
dateString := fmt.Sprintf("%d-%d-%d %d:%d:%d",
	time.Now().Year(), time.Now().Month(), time.Now().Day(),
	time.Now().Hour(), time.Now().Minute(), time.Now().Second())
```

sleep 10 秒的写法: `time.Sleep(time.Second * time.Duration(10))`

获取上个月的日期范围, 形如`2018-01-01 00:00:00`和`2018-01-01 23:55:00`.

```go
lastMonth := time.Now().Month() - 1
year := time.Now().Year()
timestart := time.Date(year, lastMonth, 1, 0, 0, 0, 0, time.Local).Format("2006-01-02 15:04:05")
timeend := time.Date(year, time.Now().Month(), 1, 0, 0, 0, 0, time.Local).Add(-time.Second * 300).Format("2006-01-02 15:04:05")
```

## go 的一种语法糖...

第一个用法主要是用于函数有多个不定参数的情况, 可以接受多个不确定数量的参数;

第二个用法是 slice 可以被打散进行传递.

```go
func test1(args ...string) { // 接受任意个string参数
	for _, v := range args {
		fmt.Println(v)
	}
}

func main() {
	test1("11", "22")

	slice1 := []string{
		"aa",
		"bb",
	}

	test1(slice1...)
}
```

输出:

```txt
[]string
11
22
[]string
aa
bb
```

## error 类型

Go 的 error 类型是一个接口:

```go
type error interface{
  Error() string
}
```

所以任何实现了`Error() string`方法签名的类型都是`error`类型.

可以使用`errors.New("im an error type.")`来创建一个 error 类型的字符串.

## make 和 new

`make`是专门用来创建`slice、map、channel`的值的, 它返回的是被创建的值,
并且立即可用;

`new`是申请一小块内存并标记它是用来存放某个值的.
它返回的是指向这块内存的`指针`, 而且这块内存并不会被初始化.
或者说对于一个引用类型的值, 那块内存虽然已经有了,
但还没法用(因为里面没有针对那个值的数据结构).

所以, 要创建引用类型的值, 不要用`new`, 能用`make`就用`make`,
不能用`make`就用复合字面量来创建.

## 值类型和引用类型

Go 中的类型分为**值类型**和**引用类型**, 但是所有的**函数传参, 都是值传递**,
没有引用传递一说, 即使传的是指针, 也是传递指针地址这个值的拷贝,
当然指针指向的内存空间还是原来的地方, 有其他语言中引用传递的效果.

值类型: `int、float、bool、string、array、struct`

引用类型: `slice、map、channel、function、interface、pointer`

值类型的特点: 变量直接存储值, 内存通常在**栈**中分配.

引用类型的特点: 变量存储的是一个地址, 这个地址对应的空间里才是真正存储的值,
内存通常在**堆**中分配.

## 操作符&和\*

`&`: 取地址, 取变量的内存地址的操作符, 操作对象是普通变量.

`*`: 取值, 取指针变量指向的值的操作符, 操作对象是指针变量.

## Go 的虚拟机

Go 算是有"虚拟机", 但是编译之后它与应用程序的源码是集成在一起的,
这跟 Java 那种传统意义上的虚拟机是不同的.

## 字面量 struct{}

字面量`struct{}`代表了空的结构体类型.
这样的类型既不包含任何字段也没有任何方法.
该类型的值所需的存储空间几乎可以忽略不计.

## 如何理解陷阱或坑

大部分所谓的陷阱或者坑, 都是由于不了解语言机理而犯的错误.
用编程语言 B 的理念和哲学去理解编程语言 A 必然会出问题.

## Go 超长字符串折行的方法

```go
ErrArgsNotFound = &Errno{
	Code: "ParameterMissing",
	Message: `The input parameter "timestart" or "timeend" that ` +
		"is mandatory for processing this request is not supplied " +
		"or value was null.",
}
```

使用`+`来拼接字符串的方式来折行, 如果使用\`, 将会保留特殊字符,
比如我们为折行而自动生成的换行符.

## 空 return

```go
type Tag struct {
    Model

    Name string `json:"name"`
    CreatedBy string `json:"created_by"`
    ModifiedBy string `json:"modified_by"`
    State int `json:"state"`
}

func GetTags(pageNum int, pageSize int, maps interface {}) (tags []Tag) {
    db.Where(maps).Offset(pageNum).Limit(pageSize).Find(&tags)

    return
}
```

函数的 return 后面没有跟着返回值, 是因为在函数定义部分已经显示的声明了返回变量名,
这个变量在函数体内也可以直接使用, 因为他在一开始就被声明了.

## 方法的接收者

看如下代码, 方法接收者可以写成`(*Databases)`, 也可以写成`(d *Databases)`,
因为方法内部没有用到接收者的字段值, 所以没必要写成第二种形式.

```go
// 每一个字段代表一个mysql数据库
type Databases struct {
	IdcBWBill *gorm.DB
}

// 等价于: var DBs *Databases = &Databases{}, 相当于实例化了Databases类型.
var DBs *Databases

// 给Databases实例对象DBs赋值, 相当于初始化连接了所有的数据库(如果有多个).
func (*Databases) Init() {
	DBs = &Databases{
		IdcBWBill: GetDB("mysqldb"),
	}
}

// 关闭数据库连接
func (*Databases) Close() {
	DBs.IdcBWBill.Close()
}
```
