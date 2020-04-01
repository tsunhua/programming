# Go 编程之最佳实践

### 使用逗号 ok 模式

```go
// 普通模式
func test() error {
    //…
    value, err := pack1.Func1(param1)
    if err != nil {
        //…
        return err
    }
    //…
    return nil
}

// 逗号 ok 模式
func test() error {
    //…
    if value, err := pack1.Func1(param1); err != nil {
        //…
        return err
    }
    //…
    return nil
}
```

### 使用 defer 确保最后调用

```go
// 关闭文件
func test() {
  // open a file f
  defer f.Close()
}

// 解锁
func test() {
  mu.Lock()
  defer mu.Unlock()
}
```

### 使用闭包归总错误检测代码

```go
func main() {
	user := User{
		Name:  "Bob",
		Age:   15,
		IsVip: false,
	}
	watchVipMovie(user)
	watchVipMovie2(user)
}

type User struct {
	Name  string
	Age   int
	IsVip bool
}
// 未使用闭包归总错误检测代码
func watchVipMovie(user User) (url string, err error) {
	if !user.IsVip {
		err = errors.New("must be vip")
		return
	}
	if user.Age < 18 {
		err = errors.New("age can not less then 18")
		return
	}
	url = "http://baidu.com"
	return
}

// 使用闭包归总错误检测代码
func watchVipMovie2(user User) (url string, err error) {
	err = func() error {
		if !user.IsVip {
			return errors.New("must be vip")
		}
		if user.Age < 18 {
			return errors.New("age can not less then 18")
		}
		return nil
	}()

	url = "http://baidu.com"
	return
}

```

### 使用 Go modules 而不是 GOPATH

<table>
  <thead>
    <tr>
      <th style="text-align:left">&#x6A21;&#x5F0F;</th>
      <th style="text-align:left">&#x51FA;&#x73B0;&#x65F6;&#x95F4;</th>
      <th style="text-align:left">&#x539F;&#x7406;</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="text-align:left">GOPATH</td>
      <td style="text-align:left">2006-01-02T15:04:05Z07:00</td>
      <td style="text-align:left">go get&#x83B7;&#x53D6;&#x7684;&#x4EE3;&#x7801;&#x4F1A;&#x653E;&#x5728;$GOPATH/src&#x4E0B;&#x9762;&#xFF0C;&#x800C;go
        build&#x4F1A;&#x5728;$GOROOT/src&#x548C;$GOPATH/src&#x4E0B;&#x9762;&#x6309;&#x7167;import
        path&#x53BB;&#x641C;&#x7D22;package&#x3002;go get &#x83B7;&#x53D6;&#x7684;&#x90FD;&#x662F;&#x5404;&#x4E2A;package
        repo&#x7684;trunk/mainline&#x7684;&#x4EE3;&#x7801;</td>
    </tr>
    <tr>
      <td style="text-align:left">Vendor</td>
      <td style="text-align:left">
        <p>Go 1.5</p>
        <p>2015-08-19</p>
      </td>
      <td style="text-align:left">Go&#x7F16;&#x8BD1;&#x5668;&#x4F1A;&#x4F18;&#x5148;&#x5728;vendor&#x4E0B;&#x641C;&#x7D22;&#x4F9D;&#x8D56;&#x7684;&#x7B2C;&#x4E09;&#x65B9;&#x5305;&#xFF0C;&#x8FD9;&#x6837;&#x5982;&#x679C;&#x5F00;&#x53D1;&#x8005;&#x5C06;&#x7279;&#x5B9A;&#x7248;&#x672C;&#x7684;&#x4F9D;&#x8D56;&#x5305;&#x5B58;&#x653E;&#x5728;vendor&#x4E0B;&#x9762;&#x5E76;&#x63D0;&#x4EA4;&#x5230;code
        repo&#xFF0C;&#x90A3;&#x4E48;&#x6240;&#x6709;&#x4EBA;&#x7406;&#x8BBA;&#x4E0A;&#x90FD;&#x4F1A;&#x5F97;&#x5230;&#x540C;&#x6837;&#x7684;&#x7F16;&#x8BD1;&#x7ED3;&#x679C;&#xFF0C;&#x4ECE;&#x800C;&#x5B9E;&#x73B0;reporduceable
        build</td>
    </tr>
    <tr>
      <td style="text-align:left">Go modules (vgo)</td>
      <td style="text-align:left">
        <p>Go 1.11</p>
        <p>2018.05</p>
      </td>
      <td style="text-align:left"><b>&#x901A;&#x5E38;</b>&#x6211;&#x4EEC;&#x4F1A;&#x5728;&#x4E00;&#x4E2A;repo(&#x4ED3;&#x5E93;)&#x4E2D;&#x521B;&#x5EFA;&#x4E00;&#x7EC4;Go
        package&#xFF0C;repo&#x7684;&#x8DEF;&#x5F84;&#x6BD4;&#x5982;&#xFF1A;github.com/bigwhite/gocmpp&#x4F1A;&#x4F5C;&#x4E3A;go
        package&#x7684;&#x5BFC;&#x5165;&#x8DEF;&#x5F84;(import path)&#xFF0C;Go
        1.11&#x7ED9;&#x8FD9;&#x6837;&#x7684;&#x4E00;&#x7EC4;&#x5728;&#x540C;&#x4E00;repo&#x4E0B;&#x9762;&#x7684;packages&#x8D4B;&#x4E88;&#x4E86;&#x4E00;&#x4E2A;&#x65B0;&#x7684;&#x62BD;&#x8C61;&#x6982;&#x5FF5;:
        module&#xFF0C;&#x5E76;&#x542F;&#x7528;&#x4E00;&#x4E2A;&#x65B0;&#x7684;&#x6587;&#x4EF6;go.mod&#x8BB0;&#x5F55;module&#x7684;&#x5143;&#x4FE1;&#x606F;&#x3002;</td>
    </tr>
  </tbody>
</table>启用 Go modules 的方式：设定环境变量 `export GO111MODULE=on`

参考：

{% embed url="https://tonybai.com/2018/07/15/hello-go-module/" %}



