# 1. 基本语法

##### 字面量

整型字面量可以使用`_`来增加可读性，比如`1_000`.

字符串字面量有两种写法：

1. `""`包起来的内容，但是有特殊字符需要转义
2. ``包起来的内容

go语言的字面量是没有类型的，它可以被解释成兼容的类型，比如可以把整型字面量赋值给浮点数。

如果使用`x := 10` 这种声明赋值语句，则使用字面量的`default type`. 浮点数的默认类型是float64。

##### 基本类型

1. bool 类型
2. 整型: int8, int16, int32, int64; uint8, uint16, uint32, uint64; byte 是uint8的别名，可以和uint8互用，大家一般使用byte; int和uint比较特殊，它根据cpu的位数来决定是64位还是32位的，在没有显式转换的情况下不能和int32/int64/uint32/uint64进行赋值、比较和其他数据运算。优先使用int(why?)。
3. 浮点数: float32, float64
4. string是__基本类型__，支持`==, !=, >, >=, <, <=`
5. 类型转换：go不支持自动类型转换。其他和java类似

##### 作用域

1. 局部定义：在函数内部定义的变量只能在本函数内访问
2. 包级定义：定义在函数外部的变量整个包都可以访问；包级变量__如果首字母是大写的，则其他的包也可以访问它__

##### 声明变量

标准结构: `var 变量名 类型 = 表达式` ，go支持自动类型推导。

如果省略了表达式，则变量的值是对应的零值：

* 数值类型变量对应的零值是0，布尔类型变量对应的零值是false，字符串类型对应的零值是空字符串(string是基本类型)
* 接口或引用类型（包括slice、map、chan和函数）变量对应的零值是nil
* 数组或结构体等聚合类型对应的零值是每个元素或字段都是对应该类型的零值。

```go
var x int // 零值
var x, y int = 10, 20 // x和y都是int
var x, y = 10, "hello" // x是int, y是string
```

如果要一次声明多个变量，可以使用declaration list: 

```go
var (
  x int
  y = 20
  z int = 30
  d, e = 40, "hello"
  f, g string
)
```

还可以使用`x := 10`，不过这种类型只能在`function`内部使用，定义包变量时不可以。

##### 常量

常量表达式的值在编译器确定，而不是在运行期，因此所有常量表达式的结果都是基础类型(Java可以是任意类型)。

常量可以有类型，也可以没有类型，看例子:

```go
const typedX int64 = 10
const untypedX = 10

var y int = 34

y * typedX // 编译错误，类型不兼容
y * untypedeX // 正确，此时untypedX就是以int类型来参与运算
```

可以把go语言中的常量理解成字面量的别名(字面量可以被解释成兼容的类型)。

##### 自增自减

1. go语言只支持`i++`和`i--`，`++i`和`--i`是非法的。
2. `i++`和`i--`是语句而不是表达式，`x = i++`是非法的，这个要区别于别的语言

##### 定义类型

什么情况下需要定义类型？两个类型相同的内部结构但是却表示完全不同的概念，比如同样是64-bits的数据，int64和TimeStamp表示的是两个概念。go语言允许定义新的类型。

标准结构: `type 类型名字 底层类型`, 这里的`底层类型`可以是通过`type`定义的新类型，比如`type TimeStamp int64`和`type TS TimeStamp`

1. 新类型和底层类型虽然内部结构相同，但是是两种不同的类型，不能直接比较和运算

2. go语言提供了类型同名的方法:

   ```go
   var t1 int64 = 1
   var t2 TimeStamp = TimeStamp(t1)
   ```

3. 可以为类型定义新的方法集

   ```go
   type TimeStamp int64
   
   func (t TimeStamp) toString()string {
     return fmt.Sprintf("%dMS", t)
   }
   
   var t3 TimeStamp = 3
   t3.toString() // 调用类型的方法
   ```


##### 字符串

`len(string)`返回的是经过编码后的字节数，不是字符数。"你好，世界"[0]得到的不是"你"而是189。

字符串是不可变的，因此相同的字符串可以共享底层结构，`slice`操作是一个低代价操作。

```
name := "abcdefghijklmnopqrstuvwxyz"
name[:]   // "abcdefghijklmnopqrstuvwxyz"
name[:3]  // "abc"
name[21:] // "xyz"
name[3:7] // defg
```

字符串支持比较操作，对字节进行逐个比较，根据字节序。

##### Arrays - Too Rigid to Use Directly

```go
var x[3]int // 这奇怪的声明语法，3个元素的值都是0
var x = [3]int{1, 2, 3} // 数组字面量
// 稀疏数组的声明: x[0] = 1, x[5]=4,x[6]=6,x[10]=100,x[11]=15，其余全是0
var x = [12]int {1, 5:4, 6, 10: 100, 15} 

// 使用字面量时可以省略长度
var x = [...]int{1, 2, 3} // x是一个长度为3的数组
var x = [...]int {1, 5:4, 6, 10: 100, 15} // x是一个长度为12的数组 
```

数组可以使用`==`和`!=`来比较。

go语言把数组的长度视为类型的一部分: [3]int 和 [4]int 是两个类型，数组基本很难作为函数的参数了；这也意味着你没办法使用一个变量来指定数组长度，数组的长度必须在编译器可知。

##### Slices - 自动扩容

首先，在声明slice的时候不需要指定长度:

```go
var x = []int{1, 2, 3} // slice字面量，注意与array声明的区别
var x = []int {1, 5:4, 6, 10: 100, 15} // x是一个长度为12的slice
var x []int // 此时 x = nil
```

slice不可比较，它只能用来判断是否等于nil: `x == nil`。slice上的内置函数:

* len(x): 返回slice的元素个数
* Cap(x): 返回slice的容量， cap(x) - len(x) 得到是剩余容量
* append(x, value): 向slice末尾添加元素，当容量不够的时候回自动扩容，capacity 小于 1024 时进行 double, 超过后至少扩容25%。
* make有两种形式
  * `var x = make([]int, 5)` 创建一个len=5, cap=5的slice，此时x=[0,0,0,0,0]；append(x, 6)后x=[0, 0, 0, 0, 0, 6], cap=10
  * `var x = make([]int, 5, 24)` 创建一个len=5, cap=24的slice; append(x, 6)后x=[0,0,0,0,0,6], cap=24

###### slice表达式

可以在slice上调用slice表达式来创建新的slice，它们共享底层的存储结构，两者任何的改变都会互相影响。这里有两点需要注意：

1. slice出来的cap = origin cap - slice offset
2. 在调用append的时候，如果一方超出了cap则会触发自动扩容，但是另外一个仍然持有的是老的数据，它俩就解耦了， 不再共享数据了

```go
origin := make([]int, 0, 5)
origin = append(origin, 1, 2, 3, 4) // origin: [1, 2, 3, 4], start = 0, cap = 5
s1 := origin[:2]                    // s1: [1, 2], start = 0, cap = 5
s2 := origin[2:]                    // s2: [3, 4], start = 2, cap = 3
// 修改s1会同步修改origin和s2
s1 = append(s1, 31, 41) // origin: [1, 2, 31, 41]; s1: [1, 2, 31, 41]; s2: [31, 41]
origin = append(origin, 5) // origin: [1, 2, 31, 41, 5]; s1: [1, 2, 31, 41]; s2: [31, 41]
// Notice: origin触发扩容，不再跟s1,s2共享底层存储了
origin = append(origin, 6)
// s1和s2仍然共享存储
s1[2] = 311 // origin: [1, 2, 31, 41, 5, 6]; s1: [1, 2, 311, 41]; s2: [311, 41]
```

slice方法还有一个参数可以指定它能使用的最大的position的位置，超过这个位置它就触发自动扩容了.

```go
origin := make([]int, 0, 5)
origin = append(origin, 1, 2, 3, 4)
s1 := origin[:2:2] // cap(s1) == 2
s1 = append(s1, 31) // s1会扩容，不再跟origin共享存储了
```

Notice: array也支持slice表达式，也共享存储.

#### Map: map[key_type]value_type

1. 内置的 map 是 hash map
2. map 会自动扩容
3. 如果 map[not exist key] 的返回值是 value type 的默认值
4. map 支持逗号表达式: value, is_contains = map[key]，如果 map 包含 key，is_contains  = true, 否则就是 false
5. 如果事先知道 map 中要存放的数据大小，可以使用: `make(map[string]int, 10)`这类语法，它可以避免扩容

```go
// map is nil，map 之间是不能互相比较的，但是可以和 nil 进行比较
var nilMap map[string]int

// 字面量
litMap := map[string]int{
  "One": 1,
  "Two": 2,
  "Three": 3,
}

// 如果实现知道 map 的大小可以提前指定
count := make(map[string]int, 10)
// 如果 key 不存在，返回 value 类型的默认值
count["one"] // 0
count["one"]++ // 虽然 one 不存在，但是它会返回0，同时 put 到 map
count["two"] = 2

// 删除元素, 如果 count is nil or key is not exist, nothing happen
delete(count, "two")

// 不存在时返回默认值，那么如果某个 key 的 value 就是默认值就会有麻烦，因此 map 支持逗号表达式
value, isContains := count["three"] // value = 0, isContains = false
```

#### 结构体

结构体的声明: `type` struct_name `struct` { fields...}

```go
type Person struct {
  name string  // 注意，没有逗号
  age int
  pet string
}
```

结构体的赋值有几种方式:

```go
var p1 Person  // p1 中所有字段都是默认值
p2 := Person {} // 两种声明方式等价

// 字面量：所有的字段都要给出值，并且使用逗号分隔
var p3 = Person {
  "cxy",
  32,
  "no", // 注意逗号
}

// 字面量：使用字段名，这种情况下就没有什么规则，顺序也不重要
var p4 = Person {
  age: 32,
  name: "cxy"
}
```

匿名结构体(比如在和外部数据转换时可能会用到):

```go
var p5 struct {
  name string
  age int
  pet string
}
p5.name = "cxy"
p5.age = 32
p5.pet = "no"

// 或者声明和赋值在一起
pet := struct {
  name string
  kind string
} {
  name: "二狗",
  kind: "长颈鹿",
}
```

结构体的比较和转换(todo: 需要再学习一下):

1. 如果结构体的所有字段都是可比较的，则这个结构体也是可比较的
2. 如果两个结构体的字段名称、类型、顺序都完全一致，则可以互相转换

### 控制结构

#### 作用域

在 go 语言中，一对`{}`就定义了一个作用域，允许作用域内相同名称的变量作用域覆盖.

#### if 和大部分语言的一致，表达式不需要()

`if`值得说的一点是可以定义变量，它的作用域只在`if else`结构内有效:

```go
if n := rand.Intn(10); n == 0 {
  return n * 1
} else if n > 5 {
  return n * 2
} else {
  return n * 3
}
```

不要滥用这个特性：

```go
func do() {...}

// 可以在 if 的语句中调用任何 simple statement，但是这不是好的做法
if do(); n == 0 {
  ....
}
```

#### for: go 中唯一的循环结构

##### C-style (没有括号)

```go
for i:= 0; i < 10; i++ {
  fmt.Println(i)
}
```

##### Condition-Only (更像C中的while)

```go
i := 1
for i < 100 {
  fmt.Println(i)
  i = i * 2
}
```

##### 死循环

```go
for {
  do something
}
```

#### for-range

```go
1. 基本模式: 
for key, value := range xxx {
  .....
}

2. 如果不想要key, 可以使用下划线代替
for _, value := range xxx {
  
}

3. 如果不想要value(仅在map下有意义，如果是slice或者别的结构使用这种方式，要考虑是否选错了数据结构)
for key := range xxx {
  ......
}

```

for-range 可以用在 slice, string, array 和 map，其中 slice，string, array 的 key 就是下标。

Notice: 在 for-range string 的结构中，遍历的不是字节而是字符，按照utf-8的编码方式将连续的字节转换成32-bit的整数

```go
name := "遍历的是字符"
for i, c := range name {
  fmt.Println(i, c, string(c))
}
====================输出
0 36941 遍
3 21382 历
6 30340 的
9 26159 是
12 23383 字
15 31526 符
```

Notice: for-range 返回的 value 是原始值的拷贝，修改 for-range 中的 value 不会影响原始的值。

#### switch

几点不同：

1. 默认不gothrough，因此不需要break
2. Case 语句可以匹配多个值
3. 类似scala，go switch 支持 boolean表达式

```go
name := cxy
switch size := len(name); size {
  case 1, 2, 3, 4:
    fmt.Println("small")
  case 5, 6, 7:
    fmt.Println("big")
  case 8, 9, 10: // do nothing
  default:
    fmt.Println("unkonw")
}

switch size := len(name); {
  case size < 5:
  	fmt.Println("small")
  case size > 10: 
    fmt.Println("unkonw")
	default:
  	fmt.Println("big")
}

```

虽然 switch 中不需要使用 break，但是它支持 break，这点要注意:

```go
for ... {
  switch x {
    case 1:
    	break // 注意这里 break 的是 switch 语句，而不是 for 语句
  }
}
```

#### goto

go语言支持 goto 指令

### 函数（只支持值传递）

go的函数跟C的比较类似，分位4个部分：`func`关键字，函数名，参数，返回值；并且 return 语句的用法跟 c 也类似。不同点：

1. 类型在后: `func add(a int, b int) int { return a + b }`
2. 函数参数连续相同的类型可以写在一起: `func add(a, b int) int`

##### 1. 可变参数

go 支持可变参数，和Java类似，也必须是最后一个参数。Java是把可变参数转成了数组，Go把可变参数转成了 slice

```go
func addTo(base int, vals...int) []int {
  out := make([]int, 0, len(vals))
  for _, v := range vals {
    out = append(out, base + v)
  }
}

// 可以传递 0 个参数
addTo(3)
// 可以传任意个参数
addTo(3, 1, 2)
// 也可以传递一个 slice，但是要使用 ... 语法
vals := []int {1, 2, 3}
addTo(3, vals...)
```

##### 2. 多返回值

```go
func multi_value() (int, int, error) = {
  return 1, 1, nil
}

// 接收多个返回值，不能加括号
id, age, error := multi_value()
// 如果某个变量不想要，可以使用下划线
id, _, error := multi_value()
```

返回的多个值是多个变量，必须在接受的时候提供对等的变量个数，而不能像Scala一样把它当成tuple_n来用。

##### 3. Function as Value

函数也是一等公民，在Go语言中，相同参数和返回值类型就算同一种类型的函数类型：

```go
func add(i int, j int) int { return i + j}
func sub(i int, j int) int { return i - j}
func mul(i int, j int) int { return i * j}
func div(i int, j int) int { return i / j}

上面4个函数的类型都是: func(int, int) int
var opMap = map(string)func(int, int) int {
  "+": add,
  "-": sub,
  "*": mul,
  "/": div,
}
```

可以使用`type`关键字来定义函数类型，比如: `type opFunc func(string, int) int `，任何一个参数是string类型和int类型并且返回int类型的函数都是它的实例。上面的map就可以改成: `map(string) opFunc {...}`，这样可以起一个有意义的名字，起到一个文档的作用。

##### 4. 匿名函数

```go
// 匿名函数
func(i int) {
  fmt.Println(i)
}(1)
```



匿名函数真正发挥作用的是`defer`和`goroutine`

##### 5. Defer 在函数 return 语句之后调用

可以在一个函数中定义多个defer，按照Last In First Out的顺序执行。

### 指针: 不支持指针运算

##### 1. Go语言中可以返回本地变量的指针: 逃逸分析(将变量存储到heap上)

```go
func return_local_variable() *int {
  x := 10  // x此时已经不是存储在栈上了，而是存储在堆上
  // 合法的
  return &x
}
```

##### 2. 使用指针提高性能

传递指针消耗的时间是固定的，跟指针指向的类型没关系。当传递的数据较大时使用指针才是有意义的，当传递数据小于1MB时没不要使用指针。《Learning Go》的作者使用 Intel i7-8700  32GB RAM的机器，传递100B的数据耗时20ns，传递指针消耗30ns。

##### 3. slice 不是指针，它的内部有一个指针字段

```go
slice 的内部结构
+-------+-----+---------+
| array | len | capacity|
+-------+-----+---------+
    |
    V
+---+---+---+---+---+---+---+---+---+---+---+---+
|   |   |   |   |   |   |   |   |   |   |   |   | value stored
+---+---+---+---+---+---+---+---+---+---+---+---+

传递给函数的 slice 实际上是 origin slice 的一个拷贝，它们共享底层存储，但是 array, len ,capacity 都是各自的。
例1：共享底层存储
func modified_shared_store(slice []int) {
  slice[0] = 39
}
slice := make([]int, 0, 10)
slice = append(slice, 100)
slice = append(slice, 200)
slice = append(slice, 300)
fmt.Println(slice) // 打印: 100, 200, 300
modified_share_store(slice)
fmt.Println(slice) // 打印: 39, 200, 300

例2：不共享 array, len ,capacity
function not_shared_array_len_capacity(slice []int) {
  slice = append(slice, 1)
}
slice := make([]int, 0, 10)
slice = append(slice, 100)
slice = append(slice, 200)
slice = append(slice, 300)
fmt.Println(slice) // 打印: 100, 200, 300
not_shared_array_len_capacity(slice) // 在方法中的append只会修改函数实参的len，不会修改origin length
fmt.Println(slice) // 打印: 100, 200, 300

例2：扩容时，array指向新的数组
func resize_to_new_array(slice []int) {
  for i := 0; i < 10; i++ {
    slice = append(slice, i)
  }
  fmt.Println(slice) // [100, 200, 300, 1, 2, 3, 4, 5, 6, 7, 8, 9]
  slice[0] = 39      // 触发扩容了，现在已经不跟 Origin slice 共享存储了，因此这个修改origin slice看不到
}
slice := make([]int, 0, 10)
slice = append(slice, 100)
slice = append(slice, 200)
slice = append(slice, 300)
fmt.Println(slice) // 打印: 100, 200, 300
resize_to_new_array(slice) // 它的任何修改都看不到了
fmt.Println(slice) // 打印: 100, 200, 300

```

##### 10. 为了性能考虑，尽量少使用指针

在Java中，除了基础类型外，其他的对象都分配在堆上的，因此访问一个对象需要先解引用，然后再访问堆。但是在Go语言中，除了指针其他的都是分配在栈上的，struct 就是分配在栈上，这样有两个好处：1）访问更快；2）不会产生垃圾，当函数调用完成，整个栈就被回收了。

关于这一点可以理解，但是写的Go代码太少了，还需要观察。
