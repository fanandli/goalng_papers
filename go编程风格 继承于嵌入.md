### 目标与内容：

最基本的也是最好理解的，我们从面向对象的思想去思考编程抽象。（因为更贴切现实）

然后我们再看看golang是如何规避面向对象中不好的地方的，即golang自身有的一套编程抽象思路。



### 理论浅谈：

在面向对象中，主体是  **对象**

在golang的抽象思维中：**结构体类型负责对象的属性**，**接口类型负责对象的方法，接口其实是 多个类型集合**（因为只要隐式实现了接口中定义的所有方法，那么这个类型就可以转为这个接口类型。那么这个接口类型其实就是所有实现了接口中定义的方法的类型的集合，正因为如此，后面可以通过断言，再转为类型集合中任意一个类型）。

（一般的面向对象语言，结构体类型中就包含了方法。就导致了，属性和行为之间过于聚合，没有低耦合。然后再继承过来继承过去，想要理清楚这些关系难度可想而知。且接口类型是显示实现，不展开。）



### 一个实例：

一般的编程抽象思路是这样的:

定义一个结构体类型。

定义一个接口类型（是一种类型，且是一种抽象类型，抽象体现在哪里？官方说是duck类型，后面再说）

接口类型是一些方法的定义，所包含的方法没有具体的代码逻辑实现。



我们可以通过结构体类型new一个对象（在堆上分配了一个内存）

如果该结构体实现了某一个接口的所有方法，则我们说，该结构体实现了这个接口，

则**该结构体类型可以转为该接口类型**。（这里就体现了接口是一种抽象类型这个说法即duck类型）这个抽象的使用在后面会说。



具体代码事例如下：

```go
package main

import "fmt"

type book struct {
	name string
	price int
	context string
}

type display interface {
	Read() (string, error)
    GetConfig() (string, error)
}

func (b book) Read() (s string, err error){
	fmt.Printf("yes run book Read function\n")
	return b.context,nil
}

func main(){
	hh := new(book)
	hh.price = 44
	hh.name = "xiyouji"
	hh.context = "xiyouji is very fun."
	context, err := hh.Read()
	if err != nil{
		fmt.Printf("function error")
	}
	fmt.Printf(context)
}
```

目前这个例子可以看到，接口其实就是对一些方法的抽象。

上面的代码直接把interface那部分删了，照样能够运行。



目前能感受到的好处就是，对象这个概念，在golang中，被分成了结构体类型和接口类型。

我们对方法又抽象出了接口。那么我们就可以脱离于对象，直接处理接口中的各个方法，重载，嵌入等等。灵活又轻便。不需要再在对象层面上嵌入，重载各种方法了(带着属性，累赘)。



duck类型的使用还没有体现，我们可以看下面：



### 类型系统：

首先我们先定义两个 接口类型，注意哈，接口，这玩意在go中是一种`类型`。

那为啥我们要创造出一个接口类型呢？



正常的，有int string 等等各种类型，创造这些类型，可能是为了内存分配方便，可能是为了抽象方便。都有其目的。

那么interface类型的目的是啥呢？ 是为了**抽象方便，便于表达逻辑**。支持duck类型思想，从而自由表达。

#### 事例：

扯远了但也挺重要，继续回过来看我们的事例：(这个事例多了一些继承，组合......)

```go
type Painter interface{
    Paint()
}

type Clicker interface{
    Click()
}
```

然后呢：

```go
type Widget struct{
    X, Y int
}

type Label struct{
    Widget
    Text string
}

func (label Label) Paint(){
    fmt.Printf("%p:Label.Paint(%q)\n", &label, label.Text)
}
               
type ListBox struct {
	Widget          // Embedding (delegation)
	Texts  []string // Aggregation
	Index  int      // Aggregation
}

func (listBox ListBox) Paint() {
	fmt.Printf("ListBox.Paint(%q)\n", listBox.Texts)
}

func (listBox ListBox) Click() {
	fmt.Printf("ListBox.Click(%q)\n", listBox.Texts)
}

//---
type Button struct {
	Label // Embedding (delegation)
}

func (button Button) Paint() { // Override
	fmt.Printf("Button.Paint(%s)\n", button.Text)
}

func (button Button) Click() {
	fmt.Printf("Button.Click(%s)\n", button.Text)
}
               
//---              
type Button struct {
	Label // Embedding (delegation)
}

func (button Button) Paint() { // Override
	fmt.Printf("Button.Paint(%s)\n", button.Text)
}

func (button Button) Click() {
	fmt.Printf("Button.Click(%s)\n", button.Text)
}
```

我们来看一下代码逻辑：

我们实现了：`Widget`，`Label`，`ListBox`，`Button`结构体类型，

 `Label`嵌入了`Widget`类型, `ListBox`嵌入了`Widget`类型，`Button`嵌入了`Label`类型。

**Label结构体类型实现了Painter接口**，所以，可以**将Label类型转为Painter类型**。

**ListBox结构体类型实现了Painter和Clicker接口**，所以，**ListBox类型可以转为Painter类型和Clicker类型**。

**Button结构体类型实现了Painter和Clicker接口**，所以，**Button类型可以转为Painter类型和Clicker类型**。

这就是接口的使用，那为啥要让这些类型可以相互转换呢？ 

肯定是为了服务于逻辑表达，方便抽象逻辑嘛。继续看：

```go
func main() {
	label := Label{Widget{10, 70}, "Label"}
	button1 := Button{Label{Widget{10, 70}, "OK"}}
	button2 := Button{Label{Widget{50, 70}, "Cancel"}}
	listBox := ListBox{Widget{10, 40},
		[]string{"AL", "AK", "AZ", "AR"}, 0}

	for _, painter := range []Painter{label, listBox, button1, button2} {
		painter.Paint()
	}

	for _, widget := range []interface{}{label, listBox, button1, button2} {
		// 默认都实现了Painter接口，可以直接调用
		widget.(Painter).Paint()
		if clicker, ok := widget.(Clicker); ok {
			clicker.Click()
		}
	}
}
```

运行结果如下：

```go
0xc000096080:Label.Paint("Label")
ListBox.Paint(["AL" "AK" "AZ" "AR"])
Button.Paint(OK)
Button.Paint(Cancel)
0xc000096100:Label.Paint("Label")
ListBox.Paint(["AL" "AK" "AZ" "AR"])
ListBox.Click(["AL" "AK" "AZ" "AR"])
Button.Paint(OK)
Button.Click(OK)
Button.Paint(Cancel)
Button.Click(Cancel)
```



#### duck类型的使用:

首先我们实现了几个结构体实例

然后我们就for循环调用测试了，

第一个:

```go
for _, painter := range []Painter{label, listBox, button1, button2} {
		painter.Paint()
}
```

对一个Painter类型的切片进行遍历，

可以看出，几个结构体实例都可以转换为Painter类型。

**这里就可以体现出DUCK类型的意思：**

**上述的所有对象都有Painter接口中的方法实现（实现了Painter接口类型）。**

**则，这些对象就都可以变为Painter类型。**



#### 断言：

从上面我们可以知道，很多类型都可以转为一个接口类型。一个类型当然也可以转为多个接口类型。

那么，我们如何精确控制我们想要的类型呢？答案就是**断言**啦。

第二个：

```go
for _, widget := range []interface{}{label, listBox, button1, button2} {
		// 默认都实现了Painter接口，可以直接调用
		widget.(Painter).Paint()
		if clicker, ok := widget.(Clicker); ok {
			clicker.Click()
		}
	}
```

是一个interface{}类型的切片。

然后将widget**断言**为了Painter或者clicker类型，调该类型的方法。如果可以断言为Clicker类型，则进入到if里的逻辑分支。



#### 小tips:

当然我们可以使用**组合**，而不使用interface{}，代替第二个range这里的操作：(当然，label没有实现Painter接口，自然就没有实现PaintClicker接口)

```go
// 定义两种方法的组合
type PaintClicker interface {
	Painter
	Clicker
}

func main() {
	for _, widget := range []PaintClicker{listBox, button1, button2} {
		widget.Paint()
        widget.Click()
	}
}
```



##### 细节就不说啦，可以看出：

1.可以通过类型嵌入，组合，使不同类型中的**参数复用**（共享），

2.通过接口类型，组合，可以做到**方法**的继承，降低代码冗余。



**以接口类型为基础的抽象思路，将参数与方法也低耦合了，让逻辑表达更加轻便**

(建立在class类型的抽象思想，让参数和方法高耦合了)

当然了，方法是支持重载的。

（很明显的，当一个类型转换为接口类型后，只能访问接口中的方法，就不能访问比如原结构体类型中的参数了。）

### 总结：

如果类是纵向归纳同一类事物，那么接口就是横向归纳具有某些相同点的不同类事物。

当然，接口还可以实现运行时多态等等其他的。。。。
