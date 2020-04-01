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

