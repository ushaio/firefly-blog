# struct

```go
// 声明类型为myint的新数据类型,int的别名
type myint int
// 定义结构体
type Book struct {
	title  string
	author string
}

func main() {
	var a myint = 10
	fmt.Printf("a=%v, type of a=%T\n", a, a)

	var book1 Book
	book1.title = "Golang"
	book1.author = "Shai"

	printBook(book1)
	fmt.Printf("printBook-book1=%v\n", book1)

	changeBook(book1)
	fmt.Printf("changeBook-book1=%v\n", book1)

	changeBook1(&book1)
	fmt.Printf("changeBook1-book1=%v\n", book1)
}

func printBook(book Book) {
	book.title = "printBookGolang"
	fmt.Printf("book=%v\n", book)
}

func changeBook(book Book) {
	book.title = "ChangeGolang"
}

func changeBook1(book *Book) {
	book.title = "PointerGolang"
}
-------------
a=10, type of a=main.myint
book={printBookGolang Shai}
printBook-book1={Golang Shai}
changeBook-book1={Golang Shai}
changeBook1-book1={PointerGolang Shai}
```

```go
type Hero struct {
	// 首字母大写，表示该属性是对外能够访问的
	Name string
	Ad   int
	// 首字母小谢，只能类的内部访问
	level int
}

// Show方法绑定到Hero结构体
func (this Hero) Show() {
	fmt.Println("Name = ", this.Name)
	fmt.Println("Ad = ", this.Ad)
	fmt.Println("Level = ", this.level)
}

func (this *Hero) GetLevel() int {
	return this.level
}

func (this *Hero) SetName(newName string) {
	fmt.Println("SetName = ", newName)
	this.Name = newName
}

func main() {
	hero := Hero{Name: "Superman", Ad: 100, level: 999}
	hero.Show()
	fmt.Println("Hero's Level is ", hero.GetLevel())

	hero.SetName("Zhangsan")
	hero.Show()
}
-----------------
Name =  Superman
Ad =  100
Level =  999
Hero's Level is  999
SetName =  Zhangsan
Name =  Zhangsan
Ad =  100
Level =  999
```