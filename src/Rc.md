> 支持多重所有权：`Rc<T>`

`Rc<T>`会在内部维护一个用于记录值引用次数的计数器,从而确定这个值是否在被引用.（则是个强引用），只有在引用次数为0时,这个值才会被**安全地释放.**

考虑下面这种情况:

```rust
fn main() {
    let a=Cons(5,Box::new(Cons(2,Box::new(Nil))));
    let b=Cons(3, Box::new(a));
    let c=Cons(10, Box::new(a));
}


use crate::List::{Cons,Nil};
enum List {
    Cons(i32,Box<List>),
    Nil,
}

```

链表`b`和`c`共享了`a`链表,编译时会报错:

```shell
 --> src\main.rs:4:9
  |
4 |     let c=Cons(10, Box::new(a));
  |         - move occurs because `a` has type `List`, which does not implement the `Copy` trait
3 |     let b=Cons(3, Box::new(a));
  |                            - value moved here
4 |     let c=Cons(10, Box::new(a));
  |  
```

> 因为在b声明后，b持有了a的所有权.此后再尝试声明c并且让c也持有a的所有权,这时的a所有权不在a上,而是在b上。

使用`Rc::clone`来声明b:

> `Rc::clone()`不是深度拷贝,只是浅度拷贝,即只增加堆上对应数据的引用次数.

```rust
use std::rc::Rc;
fn main() {
    let a=Rc::new(Cons(5,Rc::new(Cons(2,Rc::new(Nil)))));
    let b=Cons(3, Rc::clone(&a));
    let c=Cons(10, Rc::clone(&a));
    println!("b is {:?}",b);
    println!("c is {:?}",c);
}


use crate::List::{Cons,Nil};
#[derive(Debug)]
enum List {
    Cons(i32,Rc<List>),
    Nil,
}

```

