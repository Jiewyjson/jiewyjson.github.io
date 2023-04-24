> Rustacean 

# 用Rsut 设计(模拟)简单的链表



## 知识点

1. 指针类型：&,&mut,Box,Rc,Arc,&const,*mut,NonNull
2. 所有权,借用,继承可变性,内部可变性,copy
3. 关键字:struct,enum,fn,pub,impl,use...
4. 模式匹配,泛型,解构
5. 测试,
6. unsafe:裸指针,别名,栈借用,UnsafeCell,变体variance

##  功能

O(1)的分割,合并,插入和移除

## 基本布局

```rust
pub struct List {
    head: Link,
}
enum Link {
    Empty,
    More(Box<Node>),
}

struct Node {
    elem: i32,
    next: Link,
}
```

这样子设计:

- `LinkedList` 的所有节点都分配在同一个内存类型上（堆）?
- 尾节点没有分配多余的`junk`
- 枚举形式具有`null` 指针优化?

## 注意点

1. 在`push` 方法内部，把旧节点的`head` 当成新节点的`next`时:

    1. 不能把`self.head` 的所有权转让给`next` ,这时考虑到为`link`和`Node` 实现`clone` 的`trait`

    2. 为了让链表具有更好的性能，不使用`clone` 的方法进行插入,而是使用`std::mem::repalce`，[replace in std::mem - Rust (rustwiki.org)](https://rustwiki.org/zh-CN/std/mem/fn.replace.html).

        ![](https://wyjson.com/md/v2/202304132222020.png/webp)

        

2. 不需要手动为链表执行`Drop` 来清理。

## 初版单向链表

(准确来说应该是个栈，后进先出)

```rust
use std::mem;
pub struct List {
    head: Link,
}

//#[derive(Clone)]
enum Link {
    Empty,
    More(Box<Node>),
}

//#[derive(Clone)]
struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: Link::Empty }
    }

    pub fn push(&mut self, elem: i32) {
        //let new_node=Node{elem:elem,next:self.head.clone()};
        let new_node=Box::new(Node{
            elem:elem,
            next:mem::replace(&mut self.head, Link::Empty),
        });
        self.head=Link::More(new_node);
    }

    pub fn pop(&mut self)->Option<i32>{
        match mem::replace(&mut self.head,Link::Empty){
            Link::Empty=>None,
            Link::More(node)=>{
                self.head=node.next;
                Some(node.elem)
            }   
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, Link::Empty);

        while let Link::More(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, Link::Empty);
        }
    }
}

```

## 优化版

### 使用Option

注意到，初版的`Link` 是一个枚举,表示一个节点的状态:`elem` 或者是`Empty`。这和`Option` 一致了，可以使用`Option` 替代,即`Option<Box<Node>>` 

​	使用了`Option` 之后，可以用`take` 函数替代`mem::replace` 

### 使用闭包

在`pop` 方法中,使用`map` 接收闭包参数,`map` 会对`Some(x)` 中的值进行映射，返回一个新的`Some(y)` 

### 类型改为泛型

```rust
//use std::mem;

pub struct List<T>{
    head:Link<T>,
}

type Link<T>=Option<Box<Node<T>>>;//使用别名简化长命名
struct Node<T>{
    elem:T,
    next:Link<T>,
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_node = Box::new(Node {
            elem: elem,
            //next: mem::replace(&mut self.head, None),
            next:self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<T> {
        //match mem::replace(&mut self.head, None)//使用option后,同take()函数替代mem::replace
        // match self.head.take() {
        //     None => None,
        //     Some(node) => {
        //         self.head = node.next;
        //         Some(node.elem)
        //     }
        // }
        self.head.take().map(|node|{
            self.head=node.next;
            node.elem
        })
    }
}

impl<T> Drop for List<T> {
    fn drop(&mut self) {
        //let mut cur_link = mem::replace(&mut self.head, None);
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            // cur_link = mem::replace(&mut boxed_node.next, None);
            cur_link = boxed_node.next.take();
        }
    }
}
```

### 添加peek方法

`as_ref & as_mut `[标准库](https://rustwiki.org/zh-CN/std/option/enum.Option.html#method.as_ref)

```rust

    pub fn peek(&self)->Option<&T>{
        self.head.as_ref().map(|node|{
            &node.elem
        })
    }

    pub fn peek_mut(&mut self)->Option<&mut T>{
        self.head.as_mut().map(|node|{
            &mut node.elem
        })
    }
```

### 添加迭代器

- IntoIter ---  T          拿走被迭代值的所有权
- IterMut --- &mut T  可变借用
- Iter   --- &T    不可变借用

#### IntoIter

```rust

//IntoIter 迭代器会转移所有权
pub struct IntoIter<T>(List<T>);


impl<T> List<T> {
        pub fn into_iter(self)->IntoIter<T>{
        IntoIter(self)
    }
}

impl<T>Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self)->Option<Self::Item>{
        self.0.pop()
    }
}

```

#### Iter 

这部分涉及生命周期,用到`as_ref`,[生命周期消除原则]([认识生命周期 - Rust语言圣经(Rust Course)](https://course.rs/basic/lifetime.html#生命周期消除))

```rust
pub struct Iter<'a,T>{
    next:Option<&'a Node<T>>,
}

impl<T> List<T> {
        pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_deref()}
    }
    // pub fn iter(&self) -> Iter<T> {//根据生命周期消除原则,与上面一个等价
    //     Iter { next: self.head.as_deref() }
    // }
    ////或者可以用显示生命周期消除语法'_'
    // pub fn iter(&self) -> Iter<'_, T> {
    //     Iter { next: self.head.as_deref() }
    // }
}

impl <'a,T>Iterator for Iter<'a,T> {
    type Item = & 'a T;
    fn next(&mut self)->Option<Self::Item>{
        self.next.map(|node|{
            self.next=node.next.as_deref();
            &node.elem
        })
    }
}
```

#### IterMut

```rust
pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}


impl<T> List<T> {
    
    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        IterMut { next: self.head.as_deref_mut() }
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.take().map(|node| {
            self.next = node.next.as_deref_mut();
            &mut node.elem
        })
    }
}
```

## 持久化的单向链表

todo



## 双向链表

todo
