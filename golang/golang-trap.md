# Go 编程之陷阱

## 值类型

### 数字与字符串不能直接互转

反例：

```go
s := string(97) // s == "a"
```

正例：

```go
// Integer to ASCII
s := strconv.Itoa(97) // s == "97"
s := strconv.FormatInt(97, 10) // s == "97" (decimal)

// General
s := fmt.Sprintf("%d", 97)
```

同理，字符串转化为数字也要用 `strconv` 这个包内的工具才行。

参考：

{% embed url="https://yourbasic.org/golang/convert-int-to-string/" %}

## 切片

### 切片会导致整个底层数组被锁定

切片会导致整个底层数组被锁定，底层数组无法释放内存。如果底层数组较大会对内存产生很大的压力。

```go
func main() {
    headerMap := make(map[string][]byte)

    for i := 0; i < 5; i++ {
        name := "/path/to/file"
        data, err := ioutil.ReadFile(name)
        if err != nil {
            log.Fatal(err)
        }
        headerMap[name] = data[:1]
    }

    // do some thing
}
```

解决的方法是将结果克隆一份，这样可以释放底层的数组：

```go
func main() {
    headerMap := make(map[string][]byte)

    for i := 0; i < 5; i++ {
        name := "/path/to/file"
        data, err := ioutil.ReadFile(name)
        if err != nil {
            log.Fatal(err)
        }
        headerMap[name] = append([]byte{}, data[:1]...)
    }

    // do some thing
}
```

## 迭代

### ForRange 迭代返回值先 index/key 后 value

反例：

```go
arr := []string{"a", "b", "c"}
for v := range arr {
		println(v)
}
```

正例：

```go
arr := []string{"a", "b", "c"}
for _, v := range arr {
		println(v)
}
```

要点：ForRange 循环语句中的第一个值为 key，第二个值才是 value，切记。

### ForRange 迭代中  value 地址不变

反例：

```go
slice := []int{0, 1, 2, 3}
myMap := make(map[int]*int)

for index, value := range slice {
    myMap[index] = &value
}
```

正例：

```go
slice := []int{0, 1, 2, 3}
myMap := make(map[int]*int)

for index, value := range slice {
    num := value
    myMap[index] = num
}
```

要点：ForRange 会使用同一块内存去接收循环中的 value 副本，因此迭代返回的 value 地址不变，值随迭代而变。

参考：

{% embed url="https://studygolang.com/articles/9701" %}

### map 遍历顺序不固定

反例：

```go
func main() {
    m := map[string]string{
        "1": "1",
        "2": "2",
        "3": "3",
    }

    for k, v := range m {
        println(k, v)
    }
}
```

正例：

```go
func main() {
    ks := []string{"1","2","3"}
    m := map[string]string{
        "1": "1",
        "2": "2",
        "3": "3",
    }

    for _,k := range ks {
        println(k, m[k])
    }
}
```

## 函数

### 函数参数中 interface 是引用传递

反例：

```go
func main() {
   var v Hello
   change(&v)
   fmt.Print(v)
}

type Hello struct{
	name string
}

func change(v interface{}){
   v = Hello{ name:"Bob"}
}
```

正例：

```go
func main() {
   var v Hello
   change(&v)
   fmt.Print(v)
}

type Hello struct{
	name string
}

func change(v interface{}){
	a := v.(*Hello)
	a.name = "Bob"
}
```

要点：

1. 在函数调用时，像**切片**（slice）、**字典**（map）、**接口**（interface）、**通道**（channel）这样的引用类型都是默认使用**引用传递**（即使没有显式的指出指针），不需要用指针指向他们。而数组之类的是值传递。
2. 不建议使用既是传入参数也是传出参数的参数，建议分开，最好是不使用传出参数，而是直接返回值。

参考：

{% embed url="https://learnku.com/docs/the-way-to-go/function-parameters-and-return-values/3600" %}

### 可变参数是空接口类型

当参数的可变参数是空接口类型时，传入空接口的切片时需要注意参数展开的问题。

```go
func main() {
    var a = []interface{}{1, 2, 3}

    fmt.Println(a)
    fmt.Println(a...)
}
```

不管是否展开，编译器都无法发现错误，但是输出是不同的：

```text
[1 2 3]
1 2 3
```

### 局部变量同返回值命名相同时会报错

报错内容：err is shadowed during return

反例：

```go
func main() {
	err := Foo()
	if err != nil {
		println(err.Error())
	}
}

func Foo() (err error) {
	if err := Bar(); err != nil {
		return
	}
	return
}

func Bar() error {
	return errors.New("A Error")
}
```

正例：

```go
func main() {
	err := Foo()
	if err != nil {
		println(err.Error())
	}
}

func Foo() (err error) {
	if err = Bar(); err != nil {
		return
	}
	return
}

func Bar() error {
	return errors.New("A Error")
}

```

## defer

### recover 必须在 defer 函数中运行

反例：

```go
func main() {
    recover()
    panic(1)
}

// 或
func main() {
    defer recover()
    panic(1)
}

// 或
func main() {
    defer func() {
        func() { recover() }()
    }()
    panic(1)
}
```

正例：

```go
func main() {
    defer func() {
        recover()
    }()
    panic(1)
}
```

### defer 在循环中无效

反例：

```go
func main() {
	var files = []string{"file1", "file2"}
	for _, file := range files {
		var f *os.File
		var err error
		if f, err = os.Open(file); err != nil {
			return
		}
		defer f.Close()
		// 省略一些文件操作
	}
}
```

正例：

```go
func main() {
	var files = []string{"file1", "file2"}
	for _, file := range files {
		var f *os.File
		var err error
		if f, err = os.Open(file); err != nil {
			return
		}
		// 省略一些文件操作
		f.Close()
	}
}

// 或者拆分成函数
	
func main() {
	var files = []string{"file1", "file2"}
	for _, file := range files {
		handleFile(file)
	}
}

func handleFile(file string) {
	var f *os.File
	var err error
	if f, err = os.Open(file); err != nil {
		return
	}
	defer f.Close()
}

```

要点： defer 在循环中无效，不要在循环中使用 defer。

## Go 协程（Go Routine）

处于后台的 Go 协程无法保证完成任务。

反例：

```go
func main() {
    go println("Hello")
}
```

要点：Go 协程会在 main 退出时被停止，无法保证其完成并正常退出。

## 发布

### 二进制文件不包含静态资源

使用 `go build`生成的二进制文件并不包含静态资源，如项目中使用了静态资源，而打包后在他处执行却没有附带静态资源会报错。如需要将静态资源打包进二进制文件里，可参考以下几种实现：

1. go-bindata
2. go.rice
3. esc

## 参考

{% embed url="https://chai2010.cn/advanced-go-programming-book/appendix/appendix-a-trap.html" %}

