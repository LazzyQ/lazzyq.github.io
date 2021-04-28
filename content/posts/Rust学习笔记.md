---
title: "Rust学习笔记"
date: 2021-04-29T01:58:06+08:00
draft: false
original: true
categories: 
  - "Rust"
tags: 
  - "Rust基础"
---

# 安装Rust

推荐使用rustup来安装，进入到网站[https://rustup.rs/](https://rustup.rs/)，运行下面的命令进行安装

```go
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

如果已经安装过rustup使用下面的命令进行更新

```go
rustup update
```

<!--more-->

# Cargo

rustup安装后，cargo也安装完成

## 创建新项目

```go
cargo new rust-learn
```

这条会创建一个rust-learn的目录，结构如下

```go
tree -L 2
.
├── Cargo.toml
├── src
│   └── main.rs
```

- Cargo.toml是rust的mainfest文件，类似于Java的pom.xml，Golang的 go.mod文件一样
- main.rs就是我们写代码的地方了

## 构建并运行你的程序

```go
$  cargo run
   Compiling rust-learn v0.1.0 (/Users/zengqiang96/codespace/rust-learn)
    Finished dev [unoptimized + debuginfo] target(s) in 1.60s
     Running `target/debug/rust-learn`
Hello, world!
```

从输出中可以看出，cargo构建并运行了你的程序

# Rust中的创建概念

## 创建和使用变量

```go
fn main() {
  let a_number = 10;
  let a_boolean = true;
  println!("the number is {}.", a_number);
  println!("the boolean is {}.", a_boolean)
}
```

## 可变性

在 Rust 中，变量绑定默认不可变。

```go
fn main() {
  let a_number = 10;
  println!("the number is {}.", a_number);
  a_number = 15;
  println!("and now the number is {}.", a_number);
}
```

此时运行会出现错误 

```go
$ cargo run

   Compiling rust-learn v0.1.0 (/Users/zengqiang96/codespace/rust-learn)
error[E0384]: cannot assign twice to immutable variable `a_number`
 --> src/main.rs:4:3
  |
2 |   let a_number = 10;
  |       --------
  |       |
  |       first assignment to `a_number`
  |       help: make this binding mutable: `mut a_number`
3 |   println!("the number is {}.", a_number);
4 |   a_number = 15;
  |   ^^^^^^^^^^^^^ cannot assign twice to immutable variable
```

从编译输出的结果可以看出，想让变量能修改，需要使用 mut 关键字声明变量

```go
fn main() {
  let mut a_number = 10;
  println!("the number is {}.", a_number);
  a_number = 15;
  println!("and now the number is {}.", a_number);
}
```

现在这段代码就能正常运行了

## 阴影操作

```go
fn main() {
  let a_number = 5;
  let a_number = a_number + 5;
  let a_number = a_number * 2;
  println!("the number is {}.", a_number);
}
```

在Rust中可以使用相同的变量名称进行多次声明，这种操作称为"隐藏"，这是由于新变量会隐藏上一个变量。 旧变量仍存在，但无法再于此范围内引用它。

```go
fn main() {
  let number = 5;
  let number = number + 5;
  let number = number * 2;
  println!("the number is {}.", number);
}
```

变量 number 不需要使用 mut 进行声明，因为每个操作都会创建新变量并隐藏上一个变量，因此不会发生任何变化。

## 数据类型

Rust 是一种静态类型语言，这表示编译器必须准确获知代码中每个变量的数据类型。

大多数情况下，Rust编译器可以进行类型推断，但存在多种数据类型，此时用户便须告知编译器必须要使用何种数据类型。 在这些情况下，可以使用“类型注释”。

```go
let number: u32 = "42".parse().expect("Not a number!");
```

在此示例中，我们通过直接在变量名称后面批注类型“(u32)”，告诉编译器将 number 变量设为 32 位数字。

如果去掉类型注释

```go
let number = "42".parse().expect("Not a number!");
```

此时运行就会报错

```go
$  cargo run

   Compiling rust-learn v0.1.0 (/Users/zengqiang96/codespace/rust-learn)
error[E0282]: type annotations needed
 --> src/main.rs:2:7
  |
2 |   let number = "42".parse().expect("Not a number!");
  |       ^^^^^^ consider giving `number` a type
```

### 内置数据类型

**数字**

i8、u8、i16、u16、i32、u32、i64、u64、i128、u128、isize、usize、f32、f64

isize 和 usize 类型取决于运行程序的计算机类型：如果使用 64 位体系结构，则为 64 位；如果使用 32 位体系结构，则为 32 位。 只要未有指定，上述两者即为分配给整数的默认类型。

我们对文本数字使用后缀来告诉 Rust 这些数字是哪种数据类型（例如，1u32 是作为无符号的 32 位整数的数字 1）。 如果我们不提供这些类型注释，Rust 会尝试从上下文中推断类型，当类型不明确时，默认为 i32（有符号的 32 位整数）。

**布尔型**

Rust 中的布尔值由类型 bool 表示，并具有两个可能的值：true 或 false。

**字符和字符串**

char 类型是最基本的基元类型，并由单引号指定：

```go
let c = 'z';
let z = 'ℤ';
let heart_eyed_cat = '😻';
```

一些语言将其 char 类型视为 8 位无符号整数（相当于 Rust 的 u8）。 Rust 的 char 类型是 utf-8 编码的 Unicode 码位。 这意味着它们的宽度为 32 位。

str 类型也称为“字符串切片”，大多数情况下，我们使用 &str 形式来引用这些类型。现在，大家可以将 &str 视为指向不可变字符串数据的指针。 字符串字面量的类型都是 &str。

在一些字符串非字面量的情况下，Rust 提供第二种字符串类型 String。 此类型在堆上分配。 它可以存储编译时未知的文本量。

在我们学习 Rust 的所有权和租借系统之前，我们不会全面了解 String 和 &str 之间的区别。在此之前，你可以将 String 数据视为可以随程序运行而改变的字符串数据，而 &str 是不可变的字符串数据视图，它不会随程序运行而改变。

**元组**

元组是集中到一个复合体中的不同类型值的分组。 元组具有固定长度，这表示在声明后，它们便无法增大或缩小。 元组的类型由各元素类型的序列定义。

```go
("hello", 5i32, 'c');
```

此元组具有类型签名 (&'static str, i32, char)，其中：

- `&'static str` 是第一个元素的类型。
- `i32` 是第二个元素的类型。
- `char` 是第三个元素的类型。

可以按位置访问元组元素

```go
fn main() {
  let tuple = ("hello", 5,'c');
  assert_eq!( tuple.0, "hello");
  assert_eq!( tuple.1,5);
  assert_eq!( tuple.2,'c');
}
```

assert_eq! 宏用于验证两个表达式是否相等。

### 结构和枚举

结构是多个其他类型的组合体。 与元组一样，结构的各个部分可以是不同的类型，但我们可以为每个数据片段命名，以便清楚说明值的含义。

Rust 具有三类结构，分别为： 经典结构、元组结构和单元结构。

```go
//  经典结构
struct Person {
  name: String,
  age: u8,
  likes_orange: bool,
}

// 元组结构
struct Point2D(u32, u32);

// 单元结构 
struct Uint;
```

- “经典 [C 结构](https://en.wikipedia.org/wiki/Struct_(C_programming_language))”最为常用。 其中定义的每个字段都具有名称和类型。 定义后，我们可以使用 `example_struct.field` 语法来访问它们。
- “元组结构”类似于经典结构，但其字段没有名称。 如要访问单个变量，请使用与常规元组相同的语法，即 `foo.0`、`foo.1` 等（从零开始）。
- “单元结构”最常用作标记。 我们在学习 Rust 的特征功能时，将深入了解这些结构可能有用的原因。

**结构实例化**

如要在作出定义后使用某个结构，请为每个字段指定具体值来创建该结构的实例。

```go
fn main() {
  // 实例化经典结构
  let person = Person{
    name: String::from("John"),
    likes_orange: true,
    age: 25,
  };

  // 实例化元组结构
  let origin =  Point2D(0,0);

  // 实例化单元结构
  let uint = Uint;
}
```

**枚举**

enum 关键字允许创建类型，此类型可能是几个不同的变体之一。 与结构一样，枚举变体可以包含有名称的字段、没有名称的字段，或者完全不包含字段。

## 函数

Rust 中的函数定义从 fn 开始，且在函数名称之后包含一组圆括号。 编译器通过大括号判断函数体的起始和结束位置。

```go
fn main() {
  println!("Hello, world!");
  another_function();
}

fn another_function() {
  println!("Hello from another function!");
}
```

### 向函数传递参数

is_divisible_by 函数接受两个整数作为输入值，并输出一个布尔值。

```go
fn is_divisible_by(dividend: u32, divisor: u32) -> bool {
  // If the divisor is zero, we want to return early with a `false` value
  if divisor == 0 {
    return false;
  }
  dividend % divisor == 0
}
```

- `fn`：Rust 中的函数声明关键字。
- `is_divisible_by`：函数名称。
- `(dividend: u32, divisor: u32)`：此函数的参数列表。 其声明应输入两个无符号 32 位整数值。
- `> bool`：箭头指向此函数将始终返回的值类型。

此函数体中的最后一行是不含 return 关键字的表达式：

```go
dividend % divisor == 0
```

在Rust 中，函数会始终返回代码块中的最后一个表达式 ({ ... })，因此我们无需在此处使用 return 关键字。

## 集合类型

### 数组

可以通过以下两种方式定义数组：

- 使用逗号分隔列表，并用[]括起
- 使用初始值，后跟一个分号，然后添加数组长度（用方括号括起）

```rust
fn main() {
  // a comma-separated list inside of brackets
  let weekdays = ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday", "Sunday"];

  // initialize an array of 512 elements where every element is a zero
  let byte_buffer = [0_u8; 512];
}
```

使用索引(从0开始)访问数组元素

```rust
let letters = ['a', 'b', 'c', 'd', 'e', 'f', 'g'];
println!("first element of the array: {}", letters[0]);  // prints 'a'
println!("second element of the array: {}", letters[1]); // prints 'b'
```

### 向量

与数组一样，你可以使用类型为 Vec<T> 的向量存储同一类型的多个值。 但不同点在于，向量可以随时增大或缩小。 此功能隐含在向量大小中，在编译时未知，因此 Rust 无法阻止用户访问向量中的无效位置。

你会注意到 Rust 中经常使用 <T> 语法。 这些都是泛型类型参数。 当我们写入 Vec<T> 时，指示的是由某个 T 类型组成的 Vec 类型。 nameT 通常用作我们还不知道的某种类型的类型名称。 当我们实际创建向量时，它们将具有诸如 Vec<u32> 或 Vec<String> 的具体类型。

可以使用 vec! 宏初始化向量。

```rust
fn main() {
  let three_numbers = vec![1,2,3];
  println!("Init  vector: {:?}", three_numbers);

  // the vec! macro also accepts the same syntax as the array constructor
  let ten_zeroes = vec![0; 10];
  println!("Ten zeroes: {:?}", ten_zeroes); // prints [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
}
```

注意到 println! 调用中的 {:?} 格式参数。 每当我们要出于“调试”原因打印内容时，都会使用该参数；而 {} 则用于向最终用户“显示”信息。 由于 Rust 不知道如何向最终用户表示整数向量，因此使用前一个标记会导致编译错误。

使用 Vec::new() 方法也可创建向量。 用户可以将值推送到向量末尾，以便根据需要增大向量：

```rust
fn main() {
  let mut v = Vec::new();
  v.push(5);
  v.push(6);
  v.push(7);
  v.push(8);
  println!(" {:?}", v);
}
```

同样可以pop出数据

```rust
fn main() {
  let mut v = vec![1, 2];
  let two = v.pop();
  println!("{}", two.unwrap());
}
```

还支持索引访问

```rust
fn main() {
  let mut v = vec![1, 2, 3];
  let three = v[2];
  v[1] = v[1] + 5;
  println!("{:?} {}", v, three)
}
```

### 哈希映射

类型 HashMap<K, V> 存储某个 K 类型键到某个 V 类型值的映射。

可以使用 HashMap::new 方法，再使用 HashMap::insert 方法添加元素，借此创建空哈希映射。

```rust
use std::collections::HashMap;

fn main() {
  let mut book_reviews: HashMap<String, String> = HashMap::new();
  book_reviews.insert(
    "Adventures of Huckleberry Finn".to_string(), 
    "My favorite book".to_string());

  println!("{}", book_reviews["Adventures of Huckleberry Finn"]);
}
```

注意的是.to_string() 方法调用的使用，此方法将字符串字面量 (&str) 值转换为 String。若要让哈希映射“拥有”其所含的值，而非成为引用集合（指针），此方法便可派上用场。

哈希映射可以使用引用来查询现有条目，这表示即使哈希映射的类型为 HashMap<String, String>，我们也可使用 &str 或 &String 类型查找键。

## 控制流

### if/else 表达式

与其他很多的编程语言相同的用法

```rust
if 1 == 2 {
    println!("whoops, mathematics broke");
} else {
    println!("everything's fine!");
}
```

与大多数语言不同，if 块还可充当表达式。 切记，所有分支都必须为要编译的代码返回相同类型。

```rust
fn main() {
  let formal = true;
  let greeting = if formal {
      "Good evening."
  } else {
      "Hello, friend!"
  };
  println!("{}", greeting) // prints "Good evening."
}
```

### 通过 loop 永久循环

与 Rust 中其他类型的循环（如 while 和 for）不同，如果表达式通过 break 返回值，则可使用 loop。

```rust
fn main() {
  let mut i = 1;
  let something = loop {
      i *= 2;
      if i > 100 {
          break i;
      }
  };
  assert_eq!(something, 128);
}
```

循环中的每个 break 必须具有相同类型。 如果未显式提供某些内容，break; 将返回 ()（空元组）。

### while 循环

与其他编程语言的while循环使用方式差不多

```rust
fn main() {
  let mut counter = 0;
  while counter < 10 {
      println!("hello");
      counter = counter + 1;
  }
}
```

### for循环

for 表达式从迭代器中提取值。 此表达式会持续循环，直到迭代器变空为止。

```rust
fn main() {
  let a = [10, 20, 30, 40, 50];

  for element in a.iter() {
      println!("the value is: {}", element);
  }
}
```

# 错误处理

panic 是 Rust 中最简单的错误处理机制。

你可以使用 `panic!` 宏来使当前线程 panic。 它将输出一条错误消息，展开并清理堆栈，然后退出程序。

```rust
fn main() {
  panic!("Farewell!");
}
```

Rust 在执行某些操作（例如被零除或试图访问数组、矢量或哈希映射中不存在的索引）时崩溃

### 使用 Option 类型来处理缺失

Rust 标准库提供了在可能缺少值的情况下可以使用的 Option<T> 枚举。

在许多其他语言中，这将使用 null 或 nil 进行建模，但 Rust 不会在使用其他语言互操作的代码之外使用 null。

Option<T> 将自身列为两个变体之一：

```rust
enum Option<T> {
    None,     // The value doesn't exist
    Some(T),  // The value exists
}
```

None 和 Some 不是类型，而是 Option<T> 类型的变体。 这表示在其他功能中，函数不能使用 Some 或 None 作为参数，而只能使用 Option<T> 作为参数。

```rust
fn main() {
  let fruits = vec!["banana", "apple", "coconut", "orange", "strawberry"];

  // pick the first item:
  let first = fruits.get(0);
  println!("{:?}", first);

  // pick the third item:
  let third = fruits.get(2);
  println!("{:?}", third);

  // pick the 99th item, which is non-existent:
  let non_existent = fruits.get(99);
  println!("{:?}", non_existent);
}
```

输出结果

```rust
Some("banana")
Some("coconut")
None
```

### 模式匹配

match 可以使用它将值与一系列模式进行比较，然后根据哪种模式匹配来执行代码。

```rust
fn main() {
  let fruits = vec!["banana", "apple", "coconut", "orange", "strawberry"];
  for &index in [0, 2, 99].iter() {
      match fruits.get(index) {
          Some(fruit_name) => println!("It's a delicious {}!", fruit_name),
          None => println!("There is no fruit! :("),
      }
  }
}
```

Rust 将这些分支称为“match arm”，每个 arm 可以处理匹配值的一个可能结果。

可以进一步细化 match 表达式，以根据 Some 变体中的值执行不同的操作。 例如，你可以通过运行以下命令来强调椰子很棒这个事实：

```rust
fn main() {
  let fruits = vec!["banana", "apple", "coconut", "orange", "strawberry"];
  for &index in [0, 2, 99].iter() {
    match fruits.get(index) {
        Some(&"coconut") => println!("Coconuts are awesome!!!"),
        Some(fruit_name) => println!("It's a delicious {}!", fruit_name),
        None => println!("There is no fruit! :("),
    }
  }
}
```

当字符串值为 `"coconut"` 时，将匹配第一个 arm，然后使用它来确定执行流。

当你使用 match 表达式时，请记住以下规则：

- 按照从上到下的顺序对 `match` arm 进行评估。 必须在一般事例之前定义具体事例，否则它们将无法进行匹配和评估。
- `match` arm 必须涵盖输入类型可能具有的每个可能值。 如果你尝试根据非详尽模式列表进行匹配，则会出现编译器错误。

### if let 表达式

Rust 提供了一种方便的方法来测试某个值是否符合单个模式。

考虑以下示例，该示例针对 `Option<u8>` 值进行匹配，但希望仅当值为 7 时才执行代码。

```rust
let some_number: Option<u8> = Some(7);
match some_number {
    Some(7) => println!("That's my lucky number!"),
    _ => {},
}
```

我们想对 Some(7) 匹配项执行某个操作，但忽略其他 Some<u8> 值或 None 变量。 你可以在所有其他模式之后添加 _（下划线）通配符模式，以匹配任何其他项，并使用它来满足编译器耗尽 match arm 的需求。

可以使用 if let 表达式以更简短的形式编写此代码。 以下代码的行为与前面的代码相同：

```rust
if let Some(7) = some_number {
    println!("That's my lucky number!");
}
```

### 使用 unwrap 和 expect

你可以尝试使用 unwrap 方法直接访问 Option 类型的内部值。 但是要小心，因为如果变体是 None，则此方法将会 panic。

```rust
let gift = Some("candy");
assert_eq!(gift.unwrap(), "candy");
```

expect 方法的作用与 unwrap 相同，但它提供由第二个参数提供的自定义 panic 消息：

```rust
let a = Some("value");
assert_eq!(a.expect("fruits are healthy"), "value");

let b: Option<&str> = None;
b.expect("fruits are healthy"); // panics with `fruits are healthy`
```

因为这些函数可能会崩溃，所以不建议使用它。 请改为考虑使用下列方法之一：

- 使用模式匹配并显式处理 `None` 案例。
- 调用类似的非 panic 方法，例如 `unwrap_or`。如果变体为 `None`，则该方法会返回默认值；如果变体为 `Some(value)`，则会返回内部值。

```rust
assert_eq!(Some("dog").unwrap_or("cat"), "dog");
assert_eq!(None.unwrap_or("cat"), "cat");
```

## 使用 Result 类型来处理错误

Rust 提供了用于返回和传播错误的 Result<T, E> 枚举。 按照惯例，Ok(T) 变量表示成功并包含一个值，而变量 Err(E) 表示错误并包含一个错误值。

```rust
enum Result<T, E> {
    Ok(T):  // A value T was obtained.
    Err(E): // An error of type E was encountered instead.
}
```

与描述缺少某个值的可能性的 Option 类型不同，Result 类型最适合在预期会失败时使用。

`Result` 类型还具有 `unwrap` 和 `expect` 方法，这些方法执行以下操作之一：

- 如果是这种情况，则返回 `Ok` 变量中的值。
- 如果变体是 `Err`，则导致程序 panic。

# 了解 Rust 如何管理内存

# 泛型和特征

泛型类型时，可以指定所需操作，而不必考虑定义类型持有的某些内部类型。

若要实现新的泛型类型，必须在结构名称之后的尖括号内声明类型参数的名称。 然后，可以使用结构定义中的泛型类型，否则，我们将指定具体的数据类型。

```rust
struct Point<T> {
      x: T,
      y: T,
  }

  fn main() {
    let boolean = Point { x: true, y: false };
    let integer = Point { x: 1, y: 9 };
    let float = Point { x: 1.7, y: 4.3 };
    let string_slice = Point { x: "high", y: "low" };
}
```

和其他语言的泛型使用方式差不多

## 使用特征定义共享行为

特征是一组类型可实现的通用接口。 Rust 标准库有许多有用的特征，例如：

- `io::Read` 表示可以从源中读取字节的值。
- `io::Write` 表示可以写出字节的值。
- `Debug` 表示可在控制台中使用“{:?}” 格式说明符打印的值。
- `Clone` 表示可以在内存中显式复制的值。
- `ToString` 表示可以转换为 `String` 的值。
- `Default` 表示具有合理默认值的类型，例如数字为零，矢量为空，`String` 为“”。
- `Iterator` 表示可生成值序列的类型。

每个特征定义都是为未知类型定义的方法的集合，通常表示其实现器可以执行的功能或行为。

特征和其他语言中的接口类似

为了表示“具有二维区域”的概念，我们可以定义以下特征：

```rust
trait Area {
    fn area(&self) -> f64;
}
```

使用 trait 关键字和特征名称（在本例中为 Area）来声明特征。

创建一些新类型来实现 Area 特征：

```rust
struct Circle {
    radius: f64,
}

struct Rectangle {
    width: f64,
    height: f64,
}

impl Area for Circle {
    fn area(&self) -> f64 {
    use std::f64::consts::PI;
    PI * self.radius.powf(2.0)
    }
}

impl Area for Rectangle {
    fn area(&self) -> f64 {
    self.width * self.height
    }
}
```

## 使用派生特征

### 泛型类型的缺点

请看下面的代码示例：

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = Point { x: 4, y: -3 };

    if p1 == p2 { // can't compare two Point values!
        println!("equal!");
    } else {
        println!("not equal!");
    }

    println!("{}", p1); // can't print using the '{}' format specifier!
    println!("{:?}", p1); //  can't print using the '{:?}' format specifier!

}
```

三种原因会导致前面的代码失败。 请看此输出：

```rust
error[E0277]: `Point` doesn't implement `std::fmt::Display`
      --> src/main.rs:10:20
       |
    10 |     println!("{}", p1);
       |                    ^^ `Point` cannot be formatted with the default formatter
       |
       = help: the trait `std::fmt::Display` is not implemented for `Point`
       = note: in format strings you may be able to use `{:?}` (or {:#?} for pretty-print) instead
       = note: required by `std::fmt::Display::fmt`
       = note: this error originates in a macro (in Nightly builds, run with -Z macro-backtrace for more info)

    error[E0277]: `Point` doesn't implement `Debug`
      --> src/main.rs:11:22
       |
    11 |     println!("{:?}", p1);
       |                      ^^ `Point` cannot be formatted using `{:?}`
       |
       = help: the trait `Debug` is not implemented for `Point`
       = note: add `#[derive(Debug)]` or manually implement `Debug`
       = note: required by `std::fmt::Debug::fmt`
       = note: this error originates in a macro (in Nightly builds, run with -Z macro-backtrace for more info)

    error[E0369]: binary operation `==` cannot be applied to type `Point`
      --> src/main.rs:13:11
       |
    13 |     if p1 == p2 {
       |        -- ^^ -- Point
       |        |
       |        Point
       |
       = note: an implementation of `std::cmp::PartialEq` might be missing for `Point`

    error: aborting due to 3 previous errors#+end_example
```

此代码编译失败，因为我们的 `Point` 类型没有实现以下特征：

- `Debug` 特征，允许使用 `{:?}` 格式说明符来设置类型的格式，在面向程序员的调试上下文中使用。
- `Display` 特征，允许使用 `{}` 格式说明符来设置类型的格式，与 `Debug` 类似。 但 `Display` 更适合面向用户的输出。
- `PartialEq` 特征，允许比较实现器是否相等。

### 使用 derive

幸运的是，Rust 编译器可以使用 #[derive(Trait)] 属性自动为我们实现 Debug 和 PartialEq 特征，前提是它的每个字段都实现特征：

```rust
#[derive(Debug, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}
```

因为 Rust 的标准库不提供 Display 特征的自动实现，因为这面向的是最终用户。所以仍然不能完成编译。

尽管如此，我们也可以自行为类型实现 Display 特征：

```rust
use std::fmt;

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
    write!(f, "({}, {})", self.x, self.y)
    }
}
```

## 使用特征边界和泛型函数

假设我们要编写一个 Web 应用程序，并且想要一个用于将值序列化为 JSON 格式的接口。 我们可以编写如下所示的特征：

```rust
trait AsJson {
    fn as_json(&self) -> String;
}
```

然后，可以编写一个函数，该函数接受任何实现 AsJson 特征的类型。 其编写形式为 impl 后跟一组特征边界。

```rust
fn send_data_as_json(value: &impl AsJson) {
    println!("Sending JSON data to server...");
    println!("-> {}", value.as_json());
    println!("Done!\n");
}
```

我们指定 `impl` 关键字和特征名称，而不是 `item` 参数的具体类型。 此参数接受任何实现指定特征的类型。 由于该函数对将要接收的具体类型一无所知，因此只能使用匿名类型参数的特征边界可用的方法。

编写同一函数但语法略有不同的另一种方法会显式告知 T 是必须实现 `AsJson` 特征的泛型类型：

```rust
fn send_data_as_json<T: AsJson>(value: &T) { ... }
```

然后，可以声明类型并为其实现 AsJson 特征：

```rust
struct Person {
    name: String,
    age: u8,
    favorite_fruit: String,
}

struct Dog {
    name: String,
    color: String,
    likes_petting: bool,
}

impl AsJson for Person {
  fn as_json(&self) -> String {
    format!(
        r#"{{ "type": "person", "name": "{}", "age": {}, "favoriteFruit": "{}" }}"#,
        self.name, self.age, self.favorite_fruit
    )
  }
}

impl AsJson for Dog {
  fn as_json(&self) -> String {
    format!(
        r#"{{ "type": "dog", "name": "{}", "color": "{}", "likesPetting": {} }}"#,
        self.name, self.color, self.likes_petting
    )
  }
}
```

现在，Person 和 Dog 都实现了 AsJson 特征，我们可以将其用作 send_data_as_json 函数的输入参数。

```rust
fn main() {
  let laura = Person {
    name: String::from("Laura"),
    age: 31,
    favorite_fruit: String::from("apples"),
  };

  let fido = Dog {
    name: String::from("Fido"),
    color: String::from("Black"),
    likes_petting: true,
  };

  send_data_as_json(&laura);
  send_data_as_json(&fido);
}
```

## 使用迭代器

在 Rust 中，所有迭代器都会实现名为 `Iterator` 的特征，该特征在标准库中定义，并用于通过集合（如范围、数组、矢量和哈希映射）实现迭代器。

该特征的核心如下所示：

```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

Iterator 具有方法 next，调用时它将返回 Option<Item>。 只要有元素，next 方法就会返回 Some(Item)。 用尽所有元素后，它将返回 None 以指示迭代已完成。

请注意，此定义使用一些新语法：type Item 和 Self::Item，该语法使用此特征定义关联的类型。 这意味着，Iterator 特征的每一次实现还需要定义关联的 Item 类型，该类型用作 next 方法的返回类型。 换句话说，Item 类型将是从 for 循环块内的迭代器返回的类型。

### 实现自己的迭代器

创建自己的迭代器涉及两个步骤：

1. 创建一个结构来保留迭代器的状态。
2. 实现该结构的迭代器。

我们来创建一个名为 Counter 的迭代器，该迭代器从一到任意数进行计数，在创建 Counter 结构时定义。

首先，我们创建将保留迭代器状态的结构。 我们还实现了 new 方法来控制其启动方式。

```rust
#[derive(Debug)]
struct Counter {
    length: usize,
    count: usize,
}

impl Counter {
  fn new(length: usize) -> Counter {
    Counter {
        count: 0,
        length,
    }
  }
}
```

然后，我们为 Counter 结构实现 Iterator 特征。 我们将通过 usize 进行计数，因此，我们声明相关 Item 类型应为该类型。

next() 方法是唯一应定义的必需方法。 在其主体中，每次调用时递增一次计数（这就是我们从零开始的原因）。 然后，查看是否已完成计数。 我们使用 Option 类型的 Some(value) 变体表示该迭代仍会产生结果，使用 None 变体表示该迭代应停止。

```rust
impl Iterator for Counter {
  type Item = usize;

  fn next(&mut self) -> Option<Self::Item> {
    self.count += 1;
    if self.count <= self.length {
        Some(self.count)
    } else {
        None
    }
  }
}
```

我们可以通过显式调用其 next 函数来检查 Counter 是否正常工作。

```rust
fn main() {
    let mut counter = Counter::new(6);
    println!("Counter just created: {:#?}", counter);

    assert_eq!(counter.next(), Some(1));
    assert_eq!(counter.next(), Some(2));
    assert_eq!(counter.next(), Some(3));
    assert_eq!(counter.next(), Some(4));
    assert_eq!(counter.next(), Some(5));
    assert_eq!(counter.next(), Some(6));
    assert_eq!(counter.next(), None);
    assert_eq!(counter.next(), None);  // further calls to `next` will return `None`
    assert_eq!(counter.next(), None);

    println!("Counter exhausted: {:#?}", counter);
}
```

但以这种方式调用 next 会有所重复。 通过 Rust，可以在实现 Iterator 特征的类型中使用 for 循环，因此我们来执行以下操作：

```rust
fn main() {
  for number in Counter::new(10) {
    println!("{}", number);
  }
}
```

# 探索模块、包和第三方箱

在开始之前，请务必在 Rust 程序中说明代码组织背后的概念：

- **包：**
    - 是可提供一组功能的一个或多个 *箱*。
    - 包含一个 `Cargo.toml` 文件，该文件介绍了这些 *箱* 的构建方式。
- **箱：**
    - 是编译单元，即 Rust 编译器可以运行的最小代码量。
    - 编译完成后，系统将生成可执行文件或库文件。
    - 其中包含未命名的隐式顶层 *模块*。
- **模块：**
    - 是 *箱* 内的代码组织单位 *（或为嵌套形式）*。
    - 可以具有跨其他模块的递归定义。

## 程序包

每当运行 $ cargo new <project-name> 命令时，Cargo 将为我们创建一个包：

这里我们会得到一个仅包含 src/main.rs 的包，这意味着其中仅包含名为 my-project 的二进制箱：

```rust
my-project
├── src
│  └── main.rs
└── Cargo.toml
```

通过在 `src/bin` 目录中放置多个文件，包中可以包含多个二进制箱。 每个文件都是单独的二进制箱。

如果包中包含 `src/main.rs` 和 `src/lib.rs`，则其中有两个箱：库文件和二进制文件。 它们的名称与包相同。

## **板条箱**

Rust 的编译模型集中在名为 *箱* 的项目中，你可将这些项目编译为二进制文件或库文件。

使用 `cargo new` 命令创建的每个项目本身都是箱。 可在项目中用作依赖项的所有第三方 Rust 代码也是单个箱。

## **库文件箱**

我们已经介绍了如何创建二进制程序，创建库的过程也一样简单。 若要创建库，请将 `--lib` 命令行参数传递给 `cargo new` 命令：

```rust
$ cargo new --lib my-library
     Created library `my-library` package
```

你会发现你获得了一个src/lib. rs 的文件，而不是 src/main.rs 的文件。

```rust
my-library
├── src
│  └── lib.rs
└── Cargo.toml
```

当你使用 Cargo 编译此箱时，你将获得一个名为 libmy_library.rlib 的库文件，你可将该文件发布并链接到其他项目。

## 模块

Rust 具有功能强大的模块系统，该系统可以分层方式将代码拆分为逻辑单元，从而提高其可读性和重用性。

模块是项的集合：

- 常量
- 类型别名
- 函数
- 结构
- 枚举
- Traits
- `impl` 块
- 其他模块

模块还负责控制项的 *隐私*，即项是否可由外部代码（*公用*）使用，或者是否为内部实现详细信息，不可对外使用（*专用*）。

模块示例：

```rust
mod math {
    type Complex = (f64, f64);
    pub fn sin(f: f64) -> f64 { /* ... */ }
    pub fn cos(f: f64) -> f64 { /* ... */ }
    pub fn tan(f: f64) -> f64 { /* ... */ }
}

println!("{}", math::cos(45.0));
```

如果源文件中存在 mod 声明，则在运行编译器之前，系统会将模块文件的内容插入到 mod 声明在源文件中的所在位置。 换句话说，系统不会对模块进行单独编译，只会编译箱。

默认情况下，Rust 中的所有内容都是专用的，并且只能由当前模块及其后代访问。 与此相反，当项被声明为 pub 时，你可以将其视为可供外界访问。 例如：

```rust
// Declare a private struct
struct Foo;

// Declare a public struct with a private field
pub struct Bar {
    field: i32,
}

// Declare a public enum with two public variants
pub enum State {
    PubliclyAccessibleVariant,
    PubliclyAccessibleVariant2,
}
```

## 将模块分成不同的文件

当模块变得太大时，我们可能会考虑将其内容移到单独的文件中，以使代码更易于浏览。

接下来，将上述示例中的代码移动到其自己的名为 `src/authentication.rs` 的文件中，然后更改箱根文件。

```rust
mod authentication;

fn main() {
    let mut user = authentication::User::new("jeremy", "super-secret");

    println!("The username is: {}", user.get_username());
    user.set_password("even-more-secret");p
}
```

```rust
pub struct User {
    username: String,
    password_hash: u64,
}

impl User {
    pub fn new(username: &str, password: &str) -> User {
        User {
            username: username.to_string(),
            password_hash: hash_password(&password.to_owned()),
        }
    }

    pub fn get_username(&self) -> &String {
        &self.username
    }

    pub fn set_password(&mut self, new_password: &str) {
        self.password_hash = hash_password(&new_password.to_owned())
    }
}

fn hash_password<T: Hash>(t: &T) -> u64 {/* ... */}
```

当我们在 mod authentication 而不是代码块后放置分号时，编译器将使用与模块具有相同名称的其他文件加载模块的内容。

## **向项目中添加第三方箱**

若要依托管于 [crates.io](http://crates.io/) 上的库，请将其添加到 Cargo.toml 文件中：

```rust
[dependencies]
regex = "1.4.2"
```
