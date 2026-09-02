# 4.数组、切片和Map

## 数组

数组内元素只能是同类型

```go
var nameList [3]string = [3]string{"zhangsan", "lisi", "wangwu"}
fmt.Println(nameList) // [zhangsan lisi wangwu]
fmt.Printf("%v\n", nameList) // [zhangsan lisi wangwu]
fmt.Printf("%#v\n", nameList) // [3]string{"zhangsan", "lisi", "wangwu"}
```

索引

```go
// 索引
// 正向索引：从0开始
// 不支持X 负向索引：从-1开始
fmt.Printf("nameList Length: %v\n", len(nameList))
fmt.Printf("%v\n", nameList[0])
```

```go
nameList[0] = "Shai"
fmt.Printf("%v\n", nameList[0]) // Shai
```

若将数组传参给其他方法，也只是值拷贝

```go
func printArray(myArray [3]int){ // 拷贝给myArray
	...
	myArray[0] = 101 // 修改仅在该方法中生效
}
```

## 动态数组

非值传递了，而是引用传递

```go
// 动态数组
myArray := []int{1, 2, 3}
fmt.Printf("type of myArray %T\n", myArray) // type of myArray []int
dyArray(myArray)
fmt.Println(myArray)
```

```go
func dyArray(dyArray []int) {
	// _表示匿名的变量，可以不使用
	for _, value := range dyArray {
		fmt.Println("value=", value)
	}
	dyArray[0] = 101
}
---------
value= 1
value= 2
value= 3
[101 2 3]
```

## 切片（Slice）

相较数组更为灵活，其长度可变。

Slice内元素只能是同类型

```go
var nameList []string
nameList = append(nameList, "zhangsan")
nameList = append(nameList, "lisi")
nameList = append(nameList, "wangwu")

fmt.Printf("%v\n", nameList) // [zhangsan lisi wangwu]
```

```go
nameList[0] = "SHAI"
fmt.Println(nameList[0]) // SHAI
```

空数组

```go
// 声明方式1
var nullNameList []string
fmt.Println(nullNameList)
fmt.Println(nullNameList == nil) // true 零值问题
//fmt.Println(nullNameList[0]) // panic: runtime error: index out of range [0] with length 0

// 声明方式2
var nullClassList []string = []string{}
fmt.Println(nullClassList == nil) // false

// 声明方式3
nullClassList2 := []string{}
fmt.Println(nullClassList2 == nil)

// 声明方式4 （声明后再赋值）
nullClassList2 = make([]string, 5)
fmt.Println(len(nullClassList2)) // 5
```

### **make**

**分配内存并初始化底层数据结构**

```go
makeList := make([]string, 5)
fmt.Println(len(makeList)) // 5
```

```go
ageList := make([]int, 3)
fmt.Println(ageList) // [0 0 0]
// 定义一个数组
array := [3]int{1, 2, 3}
fmt.Println(array) // [1 2 3]
slices := array[:]
fmt.Println(slices) // [1 2 3]
```

**左闭右开**

| **写法** | **含义** |
| --- | --- |
| **`[:]`** | 全部 → **`start=0, end=len`** |
| **`[1:3]`** | 索引1到2 |
| **`[:3]`** | 头到索引2 |
| **`[2:]`** | 索引2到尾 |
| **`[1:3:4]`** | 索引1到2，容量限制为4 |

## 拷贝

```go
func main() {
	s := []int{1, 2, 3}
	s1 := s[0:2] // {1,2}
	fmt.Println(s1)

	s1[0] = 100 // 也会将s数组下标[0]的值改为100，因为其都是指向同一底层数组
	fmt.Println(s)
	fmt.Println(s1)

	// 如需分开指向，使用copy
	s2 := make([]int, 3)
	copy(s2, s)
	s2[0] = 120
	fmt.Println(s)
	fmt.Println(s2)
}
------------------
[1 2]
[100 2 3]
[100 2]
[100 2 3]
[120 2 3]
```

### 切片排序

```go
// 切片排序
var ints []int = []int{9, 5, 8, 2, 3, 1, 4}
// 默认升序
//sort.Ints(ints)
//fmt.Println(ints) // [1 2 3 5]

// ==================== 降序排序详解 ====================
// sort.Sort(sort.Reverse(sort.IntSlice(ints))) 理解方式：从内往外拆解
//
// 第1步（最内层）: sort.IntSlice(ints)
//   - 将 []int 转换为 sort.IntSlice 类型
//   - sort.IntSlice 实现了 sort.Interface 接口（Len, Less, Swap 三个方法）
//   - 作用：让切片具备排序能力
//
// 第2步（中间层）: sort.Reverse(...)
//   - 接收一个 sort.Interface，返回一个新的 sort.Interface
//   - 内部把 Less 方法的逻辑反转（原来返回 true 的现在返回 false）
//   - 作用：把"升序规则"变成"降序规则"
//
// 第3步（最外层）: sort.Sort(...)
//   - 接收一个 sort.Interface，执行排序
//   - 作用：按照传入的排序规则（此时是降序）进行排序
//
// 等价于伪代码：排序(反转规则(整数切片))

// ---- 分步写法（更易理解）----
intSlice := sort.IntSlice(ints)    // 第1步：切片 → 可排序接口
reversed := sort.Reverse(intSlice) // 第2步：升序 → 降序
sort.Sort(reversed)                // 第3步：执行排序
fmt.Println(ints) // [9 8 5 4 3 2 1]
//三步合一就是：sort.Sort(sort.Reverse(sort.IntSlice(ints)))
sort.Sort(sort.Reverse(sort.IntSlice(ints)))
fmt.Println(ints)
```

## Map

必须要初始化

```go
===========报错============
var aMap map[string]string
// aMap["name"] = "lisi"
fmt.Println(aMap)
--------------------------
panic: assignment to entry in nil map
```

初始化方式：

```go
// 方式1 声明Map
var map1 map[string]string
if map1 == nil {
	fmt.Println("map1 is nil")
}
// 分配数据空间，cap=10
map1 = make(map[string]string, 10)
fmt.Printf("map1, %T\n", map1)

// 方式2 声明Map
map2 := make(map[int]string)
map2[1] = "a"
map2[2] = "b"
map2[3] = "c"
map2[4] = "d"
fmt.Printf("map2=%v\n", map2)

// 方式3
var userMap map[int]string = map[int]string{
	1: "zhangsan",
	2: "lisi",
	3: "wenwu",
}
fmt.Printf("%#v\n", userMap)
```

```go
fmt.Println(userMap)
fmt.Println(userMap[1])         // zhangsan
fmt.Printf("%#v\n", userMap[9]) // ""
value1, ok := userMap[1]
fmt.Printf("%#v, %#v\n", value1, ok) // "zhangsan", true

value5, ok := userMap[5]
fmt.Printf("%#v, %#v\n", value5, ok) // "", false

// 修改value
userMap[1] = "Shai"
fmt.Printf("%#v\n", userMap[1]) // "Shai"

// 删除key
delete(userMap, 3)
fmt.Println(userMap) // map[1:Shai 2:lisi]
```

---

```go
func main() {
	cityMap := make(map[string]string)

	// 1. 增加
	cityMap["China"] = "Beijing"
	cityMap["Japan"] = "Shanghai"
	cityMap["India"] = "New Delhi"
	fmt.Printf("cityMap=%v\n", cityMap)

	// 2. 删除
	delete(cityMap, "India")
	fmt.Printf("cityMap=%v\n", cityMap)

	// 3. 修改
	cityMap["Japan"] = "Tokyo"
	fmt.Printf("cityMap=%v\n", cityMap)

	// 4. 查询
	n, ok := cityMap["Japan"]
	fmt.Println(n, ok)

	// 5. 遍历
	for key, value := range cityMap {
		fmt.Printf("key=%v, value=%v\n", key, value)
	}

	// 6. 传参
	printMap(cityMap) // 引用传递,共同指向同一个内存地址
	fmt.Printf("cityMap=%v\n", cityMap)
}

func printMap(myMap map[string]string) {
	myMap["China"] = "Shanghai"
	myMap["England"] = "London"
	fmt.Printf("myMap=%v\n", myMap)
}
--------------
cityMap=map[China:Beijing India:New Delhi Japan:Shanghai]
cityMap=map[China:Beijing Japan:Shanghai]
cityMap=map[China:Beijing Japan:Tokyo]
Tokyo true
key=China, value=Beijing
key=Japan, value=Tokyo
myMap=map[China:Shanghai England:London Japan:Tokyo]
cityMap=map[China:Shanghai England:London Japan:Tokyo]

```