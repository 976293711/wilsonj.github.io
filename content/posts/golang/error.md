---
title: "Golang Error"
date: 2020-12-04T11:29:59+08:00
draft: false
toc: false
isCJKLanguage: true
images:
tags: 
  - golang
---

### Error的几种定义

**Error are value**


#### Sentinel Error 预定义错误

- Sentinel errors 成为你 API 公共部分。
- Sentinel errors 在两个包之间创建了依赖。
- 结论: 尽可能避免 sentinel errors。


#### Error Types
Error type 是实现了 error 接口的自定义类型。
```go
type MyError struct{
    Msg string
    File string
    Line int
}

func Error() string {
    return fmt.Sprintf("%s:%d %s", e.File, e.Line, e.Msg)
}

func test() Error{
    return &MyError{"Something happened", "server.go", 42}
}
```
因为 MyError 是一个 type，调用者可以使用断言转换成这个类型，来获取更多的上下文信息。
```go
func main () {
    err := test()
    switch err:= err.(type) {
        case nil:
            //call succeeded nothine to do
        case *MyError:
            fmt.Println("error occurred on line", err.Line)
        default:
            //unknown error
    }
}
```
> 好处：与错误值相比，错误类型的一大改进是它们能够包装底层错误以提供更多上下文。

调用者要使用类型断言和类型`switch`,就要让自定义的error变为public。这种模型会导致和调用者产生强耦合，从而导致API变得脆弱。
结论：避免使用error types。 


#### Opaque errors  不透明错误

虽然你知道发生了错误，但是你没有能力看到错误的内部。作为调用者，关于操作的结果，您所知道的就是它起作用了，或者没有起作用(成功还是失败)，还有 **不建议做内容匹配**

 > 这就是不透明错误处理的全部功能–只需返回错误而不假设其内容。
```go
import "github/quux/bar"

func fn() error {
    x, err := bar.Foo()
    if err != nil {
        retuen err
    }
    // use x
}
```

还有一种情况，我们可以断言错误实现了特定的行为，而不是断言错误是特定的类型或值
```go
// temporary 暂时的
type temporary interface{
    Temporary() bool
}

func IsTemporary(err error) bool {  
    te, ok := err.(temporary)
    return ok && te.Temporary()
}
```


### Warp Error
 errors包 `github.com/pkg/errors`
  日志记录与错误无关且对调试没有帮助的信息应被视为噪音，应予以质疑。记录的原因是因为某些东西失败了，而日志包含了答案。

    1.错误要被日志记录。
    2.应用程序处理错误，保证100%完整性。
    3.之后不再报告当前错误。


​    
#### Go 1.13之后
新增了三个方法：分别是

`errors.Unwarp`    以简化处理处理包含的其他错误的错误

`errors.Is` 判断err是否属于某个错误类型

```go
//类似于
// if err == ErrNotFound{...}
if errors.Is(err, ErrNotFound) {
    //某些东西找不到了 
}
```

`errors.As` 

```go
//类似于
// if e,ok := err.(*QueryError); ok {...}
var e *QueryError
// 提示: 当前的error的类型是 *QueryError
if errors.As(err, &e) {
    // err是*QueryError类型
}
```



### GO代码中的错误 只应当处理一次（强调）

一般处理错误的方式：

1. 记录日志，然后错相关的处理。
2. 数据降级，给出默认值。




作业：我们在数据库操作的时候，比如 dao 层中当遇到一个 sql.ErrNoRows 的时候，是否应该 Wrap 这个 error，抛给上层。为什么，应该怎么做请写出代码？

答：应该，需要wrap当前错误，并且返回空值 跟 相关err出去。 等到service层 或者 controller层 处理当前错误（有可能是记录日志，有可能是直接返回默认值，即降级） 而且错误只应当处理一次。


```go
package main

import (
	stdErrors "errors"
	"fmt"
	"github.com/pkg/errors"
	"math/rand"
	"time"
)

func main() {
	logic()
}

func kitUser() (int, error) {
	rand.Seed(time.Now().Unix())
	randNum := rand.Intn(10)
	if randNum <= 5 {
		return 0, stdErrors.New("query happened error")
	} else {
		return randNum, nil
	}
}

func UserRepo() (int, error) {
	count, err := kitUser()
	if err != nil {
		return 0, errors.Wrap(err, "don't find any user")
	}
	return count, err
}

func service() (int, error) {
	count, err := UserRepo()
	return count, err
}

func logic() {
	countUser, err := service()
	if err != nil {
		fmt.Printf("original error: %T %v\n", errors.Cause(err), errors.Cause(err))
		fmt.Printf("countUser: %d err %+v \n", countUser, err)
	}
	fmt.Println("get CountUser:", countUser)
}

```








