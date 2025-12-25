### 1.值类型和引用类型变量

行为差异：
1. 在给值类型变量赋值时，一般会产生拷贝操作，且原来绑定的数据/存储空间会被覆盖。在给引用类型变量赋值时，只是改变了引用关系，原来绑定的数据/存储空间不会被覆盖。
2. 用 let 定义的变量，要求变量被初始化后都不能再赋值。对于引用类型，这只是限定了引用关系不可改变，但是所引用的数据是可以被修改的。

在仓颉编程语言中，class 和 Array 等类型属于引用类型，其他基础数据类型和 struct 等类型属于值类型。

### 2.表达式

##### “let pattern” 语法糖

构成:  `let pattern <- expression`



### 3.函数

函数是一等公民(first-class citizens)。函数可以作为函数的参数或返回值，也可以作为变量。这就是函数类型。

函数类型由参数类型和返回类型构成，两者用 `->` 连接。参数类型用 `()` 囊括，其中可以有多个参数，用 `,` 分隔。

- 示例：
  函数名:  `add`
  类型:  `(Int64, Int64) -> Int64`，表示该函数有两个参数，两个参数类型均为 `Int64`，返回类型为 `Int64` 。

```cangjie
func add(a: Int64, b: Int64): Int64 {
    a + b
}
```

### 4.Lambda 表达式

Lambda 表达式是一种匿名函数。

Lambda 表达式的语法为如下形式： `{ p1: T1, ..., pn: Tn => expressions | declarations }`。

### 5.函数调用语法糖

#### 5.1 尾随 lambda

当函数最后一个形参是函数类型，并且此时的实参是 **lambda表达式** ，可以将lambda放在函数调用的尾部。如果lambda表达式没有参数，可以省略 `=>` 例如：

```cangjie
myIf(true, { => 100 })   // General function call

myIf(true) {             // Trailing closure call
	100
}
```

函数调用只有一个lambda实参，还可以省略  `()` 。

```cangjie
func f(fn: (Int64) -> Int64) { fn(1) }

func test() {
    f { i => i * i }
}
```

#### 5.2 Flow表达式

##### 5.2.1 Pipeline 表达式  `|>`

数据流向操作符。`f(x) == x |> f`。

表达式由变量开始。

**原格式**：

```
print(substring(toUpperCase(trim(text)), 10))
```

**使用 Pipeline**：

```
text |> trim |> toUpperCase |> substring(10) |> print
```

##### 5.2.2 Composition 表达式 `~>`

两个单参函数的复合函数。`g(f(x)) == f ~> g`

表达式由函数开始。

### 6 实例xx和静态xx

以函数举例，通过 `static` 关键字修饰静态函数，并且静态函数只能由类型访问。

```cangjie
interface I {
    const func f(): Int64
    const static func f2(): Int64
}

class A <: I {
    public const func f() { 0 }
    public const static func f2() { 1 }
    const init() {}
}

const func g<T>(i: T) where T <: I {
    return i.f() + T.f2()
}

main() {
    println(g(A()))
}

```

A是I的实例，A可以访问非静态函数f( )，而静态函数f2( )只能由类型T（或者说I来访问）

### 7 结构类型

#### 7.1 普通形参和成员变量形参

既是构造函数的**参数**，同时也是这个 `struct` 的**成员变量**（属性）。

```cangjie
struct Rectangle {
    // name 是普通形参  width 和 height 是成员变量形参
    public Rectangle(name: String, let width: Int64, let height: Int64) {
        println("正在创建矩形：${name}") // OK: 可以访问普通形参
    }
}

main() {
    let rect = Rectangle("我的盒子", 10, 20)
    
    println(rect.width)  // OK: width 是成员变量
    println(rect.height) // OK: height 是成员变量
    // println(rect.name) // Error! name 不是成员变量，它在构造完后就消失了
}
```

