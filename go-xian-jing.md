# Go 陷阱

## 数字与字符串的转化

[https://yourbasic.org/golang/convert-int-to-string/](https://yourbasic.org/golang/convert-int-to-string/)

```text
s := strconv.Itoa(97) // s == "97"

var n int64 = 97
s := strconv.FormatInt(n, 10) // s == "97" (decimal)
```

## forRange 迭代

```text
for _,v := range arr{

}
```

