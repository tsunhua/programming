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

