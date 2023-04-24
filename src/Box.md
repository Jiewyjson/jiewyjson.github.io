> `Box<T>`是一个指针，`Rust` 可以确定一个`Box<T>`的大小，从而可以使用`Box<T>`即一个栈上的指针，指向堆中无法在编译时确定大小的 类型`T`.

一个`Cons`例子:

假设一个枚举类，代表一个链表节点的两种属性:一种链接下一个节点，一种是`Nil`

```rust
use crate::List::{Cons,Nil};

enum  List{
    Cons(i32,List),
    Nil,    
}
```

当`cargo build ` 时，会报错

```shell
   Compiling Box v0.1.0 (D:\rust\projects\Box)
error[E0072]: recursive type `List` has infinite size
 --> src\main.rs:7:1
  |
7 | enum  List{
  | ^^^^^^^^^^
8 |     Cons(i32,List),
  |              ---- recursive without indirection
```

这是因为`Rust` 在编译时无法确定一个`List`的大小。

这时使用`Box<T>`，因为其是一个指针，所以在编译时便可确定大小，同时它还指向了堆中存放的`List`。

```rust
fn main() {
    let list = Cons(1,Box::new(Cons(2, Box::new(Cons(3,Box::new(Nil))))));
    println!("list is {:?}",list);
}

use crate::List::{Cons, Nil};

#[derive(Debug)]
enum List {
    Cons(i32, Box<List>),
    Nil,
}

```

通过了编译。

> `Box<T>`实现了`Deref trait`,允许将`BOX<T>`的值当作引用，并且当一个`Box<T>`的值离开作用域时，由于其实现了`Drop trait` ，将导致其指向的堆中的数据也会被自动释放。



**三种解引用转换**

- 当 T:`Deref<Target=U>`时，允许`&T`转换为`&U`
- 当 T:`DerefMut<Target=U>`时，允许`&mut T`转换为`&mut U`
- 当 T:`Deref<Target=U>`时，允许`&mut T`转换为`&U`.(可变转不可变)