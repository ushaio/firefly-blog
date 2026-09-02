# channel

## channel

```go
func main() {
	// 定义一个channel
	c := make(chan int)

	go func() {
		defer fmt.Println("goroutine end")

		fmt.Println("goroutine running...")
		c <- 666 // 将 666 发送给c
	}()

	// 读取到 666
	num := <-c // 从channel c中接收数据（ <-c ），并赋值给num
	fmt.Println("num=", num)
	fmt.Println("main goruntine end")
}
----------------
goroutine running...
goroutine end
num= 666
main goruntine end
```

如果main中不从channel获取的话，`goroutine end` 无法输出

### 有缓存和无缓存

1. 当channel已满，继续向里写数据，会阻塞
2. 当channel为空，从里面取数据，也会阻塞

```go
func main() {
	c := make(chan int, 3) // channel，带有3个缓存位
	fmt.Printf("len(c)=%v, cap(c)=%v\n", len(c), cap(c))
	go func() {
		defer fmt.Println("子go程结束")
		// 0-2，顺序给chan
		for i := 0; i < cap(c)+2; i++ {
			c <- i
			fmt.Println("子go程发送元素=", i+1, "len=", len(c), "cap=", cap(c))
		}
	}()

	time.Sleep(2 * time.Second)
	for i := 0; i < cap(c); i++ {
		num := <-c
		fmt.Println("num=", num)
	}
	time.Sleep(2 * time.Second)
	fmt.Println("main 结束")
}
---------------
len(c)=0, cap(c)=3
子go程发送元素= 1 len= 1 cap= 3
子go程发送元素= 2 len= 2 cap= 3
子go程发送元素= 3 len= 3 cap= 3
num= 0
num= 1
num= 2
子go程发送元素= 4 len= 3 cap= 3
子go程发送元素= 5 len= 2 cap= 3
子go程结束
main 结束
```

## channel和range

```go
func main() {
	c := make(chan int)
	go func() {
		for i := 0; i < 5; i++ {
			c <- i
		}
		// close channel
		close(c)
	}()

	/*for {
		if data, ok := <-c; ok {
			fmt.Println("data=", data)
		} else {
			break
		}
	}*/
	// 使用range迭代来操作channel
	for data := range c {
		fmt.Println("data=", data)
	}
	fmt.Println("done")
}
-----------
data= 0
data= 1
data= 2
data= 3
data= 4
done
```

## channel和select

```go
func fibonacii(c, quit chan int) { // 简写形式,都是chan int
	x, y := 1, 1
	for {
		select {
		case c <- x: // x传给channel c
			// 如果c可写,则选case就会进来
			x = y
			y = x + y
			fmt.Println("x=", x, "y=", y)
		case <-quit: // 从channel quit单纯取值,不使用
			fmt.Println("quit")
			return
		}
	}
}
func main() {
	c := make(chan int)
	quit := make(chan int)

	// sub go
	fmt.Println("sub go start")
	go func() {
		fmt.Println("sub go func start")
		for i := 0; i < 6; i++ {
			fmt.Println("sub go", <-c)
		}
		quit <- 0
		fmt.Println("sub go func end")
	}()
	fmt.Println("sub go end")

	// main go
	fmt.Println("main go start")
	fibonacii(c, quit)
	fmt.Println("main go end")
}
---------------
sub go start
sub go end
main go start
sub go func start
sub go 1
x= 1 y= 2
x= 2 y= 4
sub go 1
sub go 2
x= 4 y= 8
x= 8 y= 16
sub go 4
sub go 8
x= 16 y= 32
x= 32 y= 64
sub go 16
sub go func end
quit
main go end
```