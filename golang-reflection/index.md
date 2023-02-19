# Go 语言反射


这篇文章讲述了Go 语言中反射(Reflection)的使用方法以及为什么要使用反射等内容.

<!--more-->

# Golang 反射学习
## 定义
从[《go程序设计语言》](https://book.itsfun.top/gopl-zh/ch12/ch12.html)了解到，在 go 语言中，反射是一种 **机制**。具体如下：
> 反射能在运行时更新变量和检查它们的值、调用它们的方法和它们支持的内在操作，而不需要在编译时就知道变量的具体类型。

从上面的描述可以大概理出如下几条结论：
   - 反射作用于某种变量，该种变量的具体类型在编译时不确定。
   - 日常使用时，当某个变量的具体类型被确定后，才能调用该具体类型的某些方法。因此，反射首先要在运行时检查到该变量的具体类型，才能进行后续操作。
   - 在编译时不确定具体类型的变量一般是**接口变量**，因此可以大概理解，反射一般是作用在接口变量上的。

## 理解
要充分理解上面几条结论，需要先理解 Go 中的**类型**和**接口**知识，下面先对类型、接口进行介绍，然后再过渡到反射的相关内容。

### 类型
我们知道 Go 是**静态编译型**语言。因此，每一个变量都有一个静态类型，也就是说每个变量的类型在编译时就已确定并固定。例如：
```golang
type MyInt int

var i int
var j MyInt
```
i 的类型为 int， j 的类型为 MyInt 。变量 i 和 j 具有不同的静态类型，尽管它们具有相同的基础类型，但如果不进行转换就不能将它们赋值给彼此。

此时大家可能有疑问，既然在编译时已经确定了变量的类型，那为何在运行时还有变量的类型能变化？这不是自相矛盾吗？别急，我们接着往下看。

### 接口
在 Go 中，接口也是一种类型。它表示一类方法的合集。
{{< admonition type=tip title="注意" open=true >}}
此处说的接口不涉及 go1.18 版本时为了支持泛型而重新定义的接口内容。因 go 的版本都是向后兼容的，因此无需担心下面的叙述在新版本 go 中不适用。
{{< /admonition >}}


在编程中，方法一般指某种行为，因此也可以说 **接口类型是对某种或某些行为的抽象和概括** 。如下，我们定义`动物`这种接口：
```golang
package main

import "fmt"

type Animal interface {
	move(int) string
}

type Human struct {
	Name string
}

func (h Human) move(n int) string {
	return fmt.Sprintf("%s 移动了 %d 步", h.Name, n)
}

type Cat struct {
}

func (c Cat) move(n int) string {
	return fmt.Sprintf("cat 移动了 %d 步", n)
}

func AnimalMove(animal Animal, n int) {
	fmt.Println(animal.move(n))
}

func main() {
	jack := Human{Name: "jack"}
	AnimalMove(jack, 3)
}

```
上面代码中，`Human` 与 `Cat` 类型都实现了 `Animal` 接口类型中的抽象方法 `move`，因此 Human 类型的变量 `jack` (以及 Cat 类型的变量)都属于 Animal 接口类型的变量(go 中接口的实现是隐式的)。

因此，当将 jack 传递给 AnimalMove 方法时，此时 jack 既属于接口变量，又是类型 Human 的变量。那么我们该如何定义 jack 呢？事实上，接口的值是这样定义的:
**接口的值是由两部分构成的，一个具体的类型，和这个具体类型对应的值**。 这两个部分被称为接口的**动态类型**和**动态值**(为了避免歧义，下文也称具体类型和具体值)。如下：

![接口值](https://cdn.staticaly.com/gh/jackbai233/image-hosting@master/20211024/接口值.54ioyxt0vag0.webp)

此时，我们再回过头理解 **Go 中变量都是静态类型的** 这句话:
```golang
// Writer is the interface that wraps the basic Write method.
type Writer interface {
    Write(p []byte) (n int, err error)
}

var x Writer  // x = nil
```
对于接口变量 x , 它的静态类型就是该接口类型(即 io.Writer)。无论 x 可能持有什么具体值(和具体类型)， x 的类型始终是 Writer 。因此它与 Go 中的变量都是静态类型的这个说法并不冲突。

接口变量的零值为nil。接口值可以使用==和!=来进行比较，它们需满足以下条件之一：

	- 两个接口值相等且它们都是nil
	- 它们的动态类型相同且其动态值能根据动态类型规则进行比较。也就是说如果接口的动态类型相同，但动态类型是不可比较的，如切片。那将它们进行比较就会产生 panic

因此，接口类型是特殊的。相较于其他类型，要么是可比较的，要么是不可比较的，因此在比较接口值或包含了接口值的 聚合类型时，必须意识到潜在的 panic。

使用fmt包的`%T`动作可以获取接口值中的**动态类型**：
```golang
var w io.Writer
fmt.Printf("%T\n", w) // "<nil>"
w = os.Stdout
fmt.Printf("%T\n", w) // "*os.File"
w = new(bytes.Buffer)
fmt.Printf("%T\n", w) // "*bytes.Buffer"
```

类型断言用来检查的接口变量值中的动态类型与要断言的类型是否匹配：
```golang
t := i.(T)
```

如果此时 i 中的动态类型不属于 T ，那么就会产生 panic。 另外可以再加一个名为 `ok` 的 bool 类型变量来避免 panic 。如果 ok 为 true，则 t 为 类型 T 的**具体值**，如果 ok 为false， 则 t 为类型 T 的零值。
需要注意的是： **接口变量中总是持有具体的动态值和动态类型，而不能持有具体的值和接口类型**：
```golang
var r io.Reader
tty, err := os.OpenFile("/dev/tty", os.O_RDWR, 0)
if err != nil {
    return nil, err
}
r = tty
// 此时接口变量r的值为tty，该值的具体类型(或动态类型)为*os.File

var w io.Writer
w = r.(io.Writer)
// 此时接口变量w的值也为tty，该值的具体类型为*os.File，并不是io.Writer，因为io.Writer是接口类型
// 类型断言用来获取一个接口变量中的具体值
```

另外， Go 中规定，所有的**具体类型的变量都能赋值给空接口变量**：
```golang
	var x any  // any是空接口的别称
	var i float64
	x = i
```

接口一般被以两种方式使用：
- 1. **重方法**：一个接口的方法表达了实现这个接口的具体类型间的相似性，但是隐藏了代码的细节和这些具体类型本身的操作。重点在于方法上，而不是具体的类型上(如文章开始的例子中 AnimalMove 方法，该方法内部只关注接口类型的 move 方法)。
- 2. **可辨识联合**：利用一个接口值可以持有各种具体类型值的能力，将这个接口认为是这些类型的联合。**类型断言**用来动态地区别这些类型，使得对每一种情况都进行不一样的处理(如类型断言、反射的使用)。

### 反射
##### 1. 基本使用
此时再看反射定义中说的*变量在运行时类型会变化*：其实说的是接口变量中随着接口值的变化，其**值中的动态类型**也在变化，也就是动态类型和动态值在变化。在基本层面上，反射只是一种检查存储在接口变量中的动态类型类型和动态值的机制。

反射是靠 Go 中的 reflect 包实行的。该包中有两个类型： Type 和 Value。这两种类型允许访问接口变量中的内容。其中有两个函数( reflect.TypeOf 和 reflect.ValueOf ）用于从接口值中获取 Type 和 Value。
```golang
package main

import (
    "fmt"
    "reflect"
)

func main() {
    var x float64 = 3.4
    fmt.Println("type:", reflect.TypeOf(x))
}
```
程序打印结果：
```
type: float64
```
[Go 文档](https://pkg.go.dev/reflect#TypeOf)中, reflect.TypeOf 的参数是一个空接口变量。因此在调用 reflect.TypeOf(x) 时， x 首先存储在一个空接口中，然后作为参数传递； reflect.TypeOf 会解压该空接口变量并获取其值中的动态类型信息。

reflect.Type 和 reflect.Value 这两种类型本身带了很多方法可以让我们检查和操作它们。一个重要的例子是 reflect.Value 的 Type 方法；另一个是 reflect.Type 和 reflect.Value 都有一个 Kind 方法，该方法返回一个常量，指示存储的值的动态类型： Uint 、 Float64 、 Slice 等等。另外， reflect.Value 上的方法如 Int 和 Float 能让我们获取存储在其中的动态值（如 int64 和 float64 ）：
```golang
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println("type:", v.Type())
fmt.Println("kind is float64:", v.Kind() == reflect.Float64)
fmt.Println("value:", v.Float())
```
打印结果：
```
type: float64
kind is float64: true
value: 3.4
```

另外，反射有以下两个特性：
- reflect.Value 的 Int 方法返回一个 int64 ，而 SetInt 入参也是 int64
- Kind 方法返回的是变量的最底层类型，而不是静态类型。如：
    ```golang
    type MyInt int
    var x MyInt = 7
    v := reflect.ValueOf(x)
    ```
    v的 Type 方法返回的是 Myint, 而 Kind 方法返回的依然是 reflect.Int。

##### 2. 从反射回到接口值
像物理反射一样，Go 中的反射也能进行逆操作。
给定一个 reflect.Value ，我们可以使用其 Interface 方法恢复成一个接口值。实际上，该方法是将动态类型和动态值信息打包回空接口并返回：
```golang
	var x float64 = 3.4
	v := reflect.ValueOf(x)
	y := v.Interface() // y will have type float64.
	fmt.Println(y)
```
##### 3. 要修改反射对象，值必须是可设置的
首先，看如下代码：
```golang
var x float64 = 3.4
v := reflect.ValueOf(x)
v.SetFloat(7.1) // Error: will panic.
```
如果运行如上代码，则会产生 panic。原因不在于 7.1 不可寻址，而是 v 不能设置(即不能寻址)，可设置性是 reflect.Value 的属性，但并非所有 reflect.Value 都拥有。reflect.Value 的 CanSet 方法可以用来检测该特性：
```golang
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println("settability of v:", v.CanSet())
// print: settability of v: false
```
**可设置性**是反射对象是否能够修改存储在反射对象的实际变量值的属性。可设置性由反射对象是否持有原始变量内容所决定。

我们知道 go 是 **值拷贝**的, 就像函数传参一样，传递给形参的值实际是实参的值的一份拷贝副本。反射中的 reflect.ValueOf 方法也是如此， 因此对副本 v 进行更改并不能更改被传入的变量 x 的原始值。这对于反射来说是没有意义的。因此反射会报 panic。

如果我们想通过反射修改 x ，就必须给反射一个指向我们要修改的值的指针。例如：
```golang
var x float64 = 3.4
p := reflect.ValueOf(&x) // Note: take the address of x.
fmt.Println("type of p:", p.Type())
fmt.Println("settability of p:", p.CanSet())
```
打印结果：
```
type of p: *float64
settability of p: false
```
此时我们很纳闷。我已经传入了 x 的指针了，为什么还不能更改。其实仔细想一下，在这里，我们的意图是为了修改 x 的值，而不是修改 x 的地址。因此 p 在这里代表的是 x 的地址，我们实际需要的是 `*p`,因此为了获取 p 实际指向的内容，需要再调用 reflect.Value 的 `Elem` 方法。它通过指针间接访问结果，并将结果保存在名为 v 的反射 reflect.Value 中：
```
v := p.Elem()
fmt.Println("settability of v:", v.CanSet())
```
现在 v 是一个可设置的反射对象，打印结果如下：
```
settability of v: true
```
因为 v 代表 x ，我们终于可以使用 v.SetFloat 来修改 x 的值了：
```golang
v.SetFloat(7.1)
fmt.Println(v.Interface())
fmt.Println(x)
```
打印结果：
```
7.1
7.1
```
另外， 如果 x 是一个结构体变量的话，如果要对结果体中的字段进行更改，那么该字段必须是可导出的(即字段首字母大写)。

以上就是关于反射的理解和基本使用了，关于反射的更多用法，请参考 [《go 程序设计语言》](https://book.itsfun.top/gopl-zh/ch12/ch12-04.html)中的示例。

码字不易，请尊重原创。如需转载，请标明出处 :smile:！

## 参考
1. [laws-of-reflection](https://go.dev/blog/laws-of-reflection)
2. [go 程序设计语言--反射](https://book.itsfun.top/gopl-zh/ch12/ch12.html)
