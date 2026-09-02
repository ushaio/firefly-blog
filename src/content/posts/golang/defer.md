# defer

```go
func main() {
	defer fmt.Println("defer world") // 之后执行
	fmt.Println("hello1")
	fmt.Println("hello2")
}
---------
hello1
hello2
defer world
```

<img src="attach/image%2011.png" alt="image.png" width="461" height="295">

# defer和return

在return之后调用

# defer 的使用场景

## 1. 资源释放（关闭文件 / 连接）

```go
f, err := os.Open("file.txt")
if err != nil {
    log.Fatal(err)
}
defer f.Close() // 函数返回前自动关闭文件
```

## 2. 释放锁

```go
mu.Lock()
defer mu.Unlock() // 忘记解锁或提前 return 也不会死锁
```

## 3. panic 恢复（recover）

```go
defer func() {
    if r := recover(); r != nil {
        fmt.Println("恢复 panic:", r)
    }
}()
```

## 4. 记录函数执行时间 / 日志

```go
func foo() {
    start := time.Now()
    defer func() { fmt.Println("耗时:", time.Since(start)) }()
    // ...
}
```

## 5. 关闭 HTTP 响应体

```go
resp, err := http.Get(url)
if err != nil {
    return err
}
defer resp.Body.Close()
```

## 注意点

- defer 按 **LIFO（后进先出）** 顺序执行
- 参数在 defer 语句执行时就**立即求值**（不是函数返回时）
- 即使发生 panic，defer 也一定会执行

&nbsp;