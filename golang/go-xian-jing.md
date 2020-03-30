# Go 编程之陷阱

## 案例1 ：数字字符串化

反例：

```text
s := string(97) // s == "a"
```

正例：

```text
// Integer to ASCII
s := strconv.Itoa(97) // s == "97"
s := strconv.FormatInt(97, 10) // s == "97" (decimal)

// General
s := fmt.Sprintf("%d", 97)
```

同理，字符串转化为数字也要用 `strconv` 这个包内的工具才行。

[了解更多](https://yourbasic.org/golang/convert-int-to-string/)

## 案例 2：ForRange 迭代

反例：

```text
arr := []string{"a", "b", "c"}
for v := range arr {
		println(v)
}
```

正例：

```text
arr := []string{"a", "b", "c"}
for _, v := range arr {
		println(v)
}
```

ForRange 循环语句中的第一个值为 key，第二个值才是 value，切记。

{% embed url="https://studygolang.com/articles/9701" %}

{% embed url="https://chai2010.cn/advanced-go-programming-book/appendix/appendix-a-trap.html" %}

## 案例3：值传递还是引用传递

{% embed url="https://learnku.com/docs/the-way-to-go/function-parameters-and-return-values/3600" %}

```text
func main() {
   var v Hello
   change2(&v)
   fmt.Print(v)
}

type Hello struct{
	name string
}

func change(v interface{}){
   v = Hello{ name:"122"}
}

func change2(v interface{}){
	a := v.(*Hello)
	a.name = "123"
}
```

## 案例 4：可变参数是空接口类型

当参数的可变参数是空接口类型时，传入空接口的切片时需要注意参数展开的问题。

```text
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

