# Go 编程之最佳实践

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

