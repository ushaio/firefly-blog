# 创建goroutine

```go
func newTask() {
	i := 0
	for {
		i++
		fmt.Printf("newTask.task i = %d\n", i)
		time.Sleep(1 * time.Second)
	}
}

func main() {
	go newTask()

	i := 0
	for {
		i++
		fmt.Printf("main.task i = %d\n", i)
		time.Sleep(1 * time.Second)
	}
}
```

## 

```go
func main() {
	// 创建承载一个形参为空,返回为空的函数
	go func() {
		defer fmt.Println("A.defer")

		//return // A.defer end

		func() {
			// 退出当前goroutine
			defer fmt.Println("B.defer")
			runtime.Goexit() // 退出线程。本行之后的代码不执行,包括defer
			fmt.Println("B")
		}() // 定义函数,并调用
	}() // 定义函数,并调用

	go func(a int, b int) bool {
		fmt.Println("a=", a, ", b=", b)
		return a > b // 如何拿到return?
	}(10, 20)

	for {
		time.Sleep(1 * time.Second)
	}
}
```