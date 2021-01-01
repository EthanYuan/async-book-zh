# Pinning

要轮询 Future ，必须使用一种称为`Pin<T>`的特殊类型，来固定 Future。如果您阅读了在上一节["Executing `Future`s and Tasks"]中，[the `Future` trait]的解释，您会发现`Pin`来自`self: Pin<&mut Self>`，它是`Future::poll`方法的定义。但这究竟是什么意思，为什么我们需要它？

## Why Pinning

`Pin`与` Unpin`标记配合使用。Pinning可以保证实现`!Unpin`的对象永远不会被移动。要了解为什么需要这样做，我们需要记住`async` /`.await`的工作方式。考虑以下代码：

```rust,edition2018,ignore
let fut_one = /* ... */;
let fut_two = /* ... */;
async move {
    fut_one.await;
    fut_two.await;
}
```

在幕后，新建一个实现`Future`的匿名类型，它提供一个`poll`(轮询)方法，看起来像这样的：

```rust,ignore
// 这个 `Future` 类型，由我们的 `async { ... }` 代码块生成而来
struct AsyncFuture {
    fut_one: FutOne,
    fut_two: FutTwo,
    state: State,
}

// 是我们 `async` 代码块可处于的，状态列表
enum State {
    AwaitingFutOne,
    AwaitingFutTwo,
    Done,
}

impl Future for AsyncFuture {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        loop {
            match self.state {
                State::AwaitingFutOne => match self.fut_one.poll(..) {
                    Poll::Ready(()) => self.state = State::AwaitingFutTwo,
                    Poll::Pending => return Poll::Pending,
                }
                State::AwaitingFutTwo => match self.fut_two.poll(..) {
                    Poll::Ready(()) => self.state = State::Done,
                    Poll::Pending => return Poll::Pending,
                }
                State::Done => return Poll::Ready(()),
            }
        }
    }
}
```

当`poll`先被调用，它将轮询`fut_one`。如果`fut_one`还未完成，`AsyncFuture::poll`将返回。 Future 对`poll`进行调用，将在上一个停止的地方继续。这个过程一直持续到 Future 能成功完成。

但是，如果我们有一个，使用引用的`async`代码块？例如：

```rust,edition2018,ignore
async {
    let mut x = [0; 128];
    let read_into_buf_fut = read_into_buf(&mut x); // &mut x
    read_into_buf_fut.await;
    println!("{:?}", x);
}
```

这会编译成什么结构？

```rust,ignore
struct ReadIntoBuf<'a> {
    buf: &'a mut [u8], // 指向下面的 `x`
}

struct AsyncFuture {
    x: [u8; 128],
    read_into_buf_fut: ReadIntoBuf<'what_lifetime?>,
}
```

在这里，`ReadIntoBuf` Future 拿着一个引用，指向我们结构的其他字段，即`x`。但是，如果`AsyncFuture`被移动(move)，`x`的位置也会移动，使存储在`read_into_buf_fut.buf`中的指针无效。

将 Future 固定到内存中的特定位置，可以避免此问题，从而可以安全地创建，对`async`代码块内部值的引用。

## Pinning in Detail

让我们尝试使用一个比较简单的示例来了解pinning。前面我们遇到的问题，最终可以归结为如何在Rust中处理自引用类型的引用的问题。 

现在，我们的示例如下所示：

```rust
use std::pin::Pin;

#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
}

impl Test {
    fn new(txt: &str) -> Self {
        Test {
            a: String::from(txt),
            b: std::ptr::null(),
        }
    }

    fn init(&mut self) {
        let self_ref: *const String = &self.a;
        self.b = self_ref;
    }

    fn a(&self) -> &str {
        &self.a
    }

    fn b(&self) -> &String {
        unsafe {&*(self.b)}
    }
}
```

`Test`提供了获取字段a和b值引用的方法。由于b是对a的引用，因此我们将其存储为指针，因为Rust的借用规则不允许我们定义这种生命周期。现在，我们有了所谓的自引用结构。 

如果我们不移动任何数据，则该示例运行良好，可以通过运行示例观察：

```rust
fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test2 = Test::new("test2");
    test2.init();

    println!("a: {}, b: {}", test1.a(), test1.b());
    println!("a: {}, b: {}", test2.a(), test2.b());

}
# use std::pin::Pin;
# #[derive(Debug)]
# struct Test {
#     a: String,
#     b: *const String,
# }
#
# impl Test {
#     fn new(txt: &str) -> Self {
#         Test {
#             a: String::from(txt),
#             b: std::ptr::null(),
#         }
#     }
#
#     // We need an `init` method to actually set our self-reference
#     fn init(&mut self) {
#         let self_ref: *const String = &self.a;
#         self.b = self_ref;
#     }
#
#     fn a(&self) -> &str {
#         &self.a
#     }
#
#     fn b(&self) -> &String {
#         unsafe {&*(self.b)}
#     }
# }
```

我们得到了我们期望的结果：

```powershell
a: test1, b: test1
a: test2, b: test2
```

让我们看看如果将`test1`与`test2`交换导致数据移动会发生什么：

```rust
fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test2 = Test::new("test2");
    test2.init();

    println!("a: {}, b: {}", test1.a(), test1.b());
    std::mem::swap(&mut test1, &mut test2);
    println!("a: {}, b: {}", test2.a(), test2.b());

}
```

我们天真的以为应该两次获得`test1`的调试打印，如下所示：

```powershell
a: test1, b: test1
a: test1, b: test1
```

但我们得到的是：

```powershell
a: test1, b: test1
a: test1, b: test2
```

`test2.b`的指针仍然指向了原来的位置，也就是现在的`test1`的里面。该结构不再是自引用的，它拥有一个指向不同对象字段的指针。这意味着我们不能再依赖`test2.b`的生命周期和`test2`的生命周期的绑定假设了。 

如果您仍然不确定，那么下面可以让您确定了吧：

```rust
fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test2 = Test::new("test2");
    test2.init();

    println!("a: {}, b: {}", test1.a(), test1.b());
    std::mem::swap(&mut test1, &mut test2);
    test1.a = "I've totally changed now!".to_string();
    println!("a: {}, b: {}", test2.a(), test2.b());

}
```

下图可以帮助您直观地了解正在发生的事情：

![](/img/swap_problem.jpg)

这很容易使它展现出未定义的行为并“壮观地”失败。

## Pinning in Practice

让我们看下Pinning和`Pin`类型如何帮助我们解决此问题。 

`Pin`类型封装了指针类型，它保证不会移动指针后面的值。例如，`Pin<&mut T>`，`Pin<&T>`，`Pin<Box<T>>`都保证`T`不被移动，除非`T:!Unpin`。 

大多数类型在移动时都没有问题。这些类型实现了`Unpin`特型。可以将`Unpin`类型的指针自由的放置到`Pin`中或从中取出。例如，`u8`是`Unpin`，因此`Pin<&mut u8>`的行为就像普通的`&mut u8`。 

但是，固定后无法移动的类型具有一个标记为`!Unpin`的标记。由async / await创建的Futures就是一个例子。

### Pinning to the Stack

回到我们的例子。我们可以使用`Pin`来解决我们的问题。让我们看一下我们的示例的样子，我们需要一个pinned的指针：

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
    _marker: PhantomPinned,
}


impl Test {
    fn new(txt: &str) -> Self {
        Test {
            a: String::from(txt),
            b: std::ptr::null(),
            _marker: PhantomPinned, // This makes our type `!Unpin`
        }
    }
    fn init<'a>(self: Pin<&'a mut Self>) {
        let self_ptr: *const String = &self.a;
        let this = unsafe { self.get_unchecked_mut() };
        this.b = self_ptr;
    }

    fn a<'a>(self: Pin<&'a Self>) -> &'a str {
        &self.get_ref().a
    }

    fn b<'a>(self: Pin<&'a Self>) -> &'a String {
        unsafe { &*(self.b) }
    }
}
```

如果我们的类型实现`!Unpin`，则将对象固定到栈始终是不安全的。您可以使用诸如[`pin_utils`][pin_utils] 之类的板条箱来避免在固定到栈时编写我们自己的不安全代码。 下面，我们将对象`test1`和`test2`固定到栈上：

```rust
pub fn main() {
    // test1 is safe to move before we initialize it
    let mut test1 = Test::new("test1");
    // Notice how we shadow `test1` to prevent it from being accessed again
    let mut test1 = unsafe { Pin::new_unchecked(&mut test1) };
    Test::init(test1.as_mut());

    let mut test2 = Test::new("test2");
    let mut test2 = unsafe { Pin::new_unchecked(&mut test2) };
    Test::init(test2.as_mut());

    println!("a: {}, b: {}", Test::a(test1.as_ref()), Test::b(test1.as_ref()));
    println!("a: {}, b: {}", Test::a(test2.as_ref()), Test::b(test2.as_ref()));
}
```

如果现在尝试移动数据，则会出现编译错误：

```rust
pub fn main() {
    let mut test1 = Test::new("test1");
    let mut test1 = unsafe { Pin::new_unchecked(&mut test1) };
    Test::init(test1.as_mut());

    let mut test2 = Test::new("test2");
    let mut test2 = unsafe { Pin::new_unchecked(&mut test2) };
    Test::init(test2.as_mut());

    println!("a: {}, b: {}", Test::a(test1.as_ref()), Test::b(test1.as_ref()));
    std::mem::swap(test1.get_mut(), test2.get_mut());
    println!("a: {}, b: {}", Test::a(test2.as_ref()), Test::b(test2.as_ref()));
}
```

类型系统阻止我们移动数据。

需要注意，栈固定将始终依赖于您在编写`unsafe`时提供的保证。虽然我们知道`&'a mut T`所指的对象在生命周期`'a`中固定，但我们不知道`'a`结束后数据`&'a mut T`指向的数据是不是没有移动。如果移动了，就违反了Pin约束。 

容易犯的一个错误就是忘记隐藏原始变量，因为您可以drop`Pin`并将数据移动到`&'a mut T`之后，如下所示（这违反了Pin约束）

```rust
fn main() {
   let mut test1 = Test::new("test1");
   let mut test1_pin = unsafe { Pin::new_unchecked(&mut test1) };
   Test::init(test1_pin.as_mut());
   drop(test1_pin);
   println!(r#"test1.b points to "test1": {:?}..."#, test1.b);
   let mut test2 = Test::new("test2");
   mem::swap(&mut test1, &mut test2);
   println!("... and now it points nowhere: {:?}", test1.b);
}
```



### Pinning to the Heap

将`!Unpin`类型固定到堆将为我们的数据提供稳定的地址，所以我们知道指向的数据在固定后将无法移动。与栈固定相反，我们知道数据将在对象的生命周期内固定。

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
    _marker: PhantomPinned,
}

impl Test {
    fn new(txt: &str) -> Pin<Box<Self>> {
        let t = Test {
            a: String::from(txt),
            b: std::ptr::null(),
            _marker: PhantomPinned,
        };
        let mut boxed = Box::pin(t);
        let self_ptr: *const String = &boxed.as_ref().a;
        unsafe { boxed.as_mut().get_unchecked_mut().b = self_ptr };

        boxed
    }

    fn a<'a>(self: Pin<&'a Self>) -> &'a str {
        &self.get_ref().a
    }

    fn b<'a>(self: Pin<&'a Self>) -> &'a String {
        unsafe { &*(self.b) }
    }
}

pub fn main() {
    let mut test1 = Test::new("test1");
    let mut test2 = Test::new("test2");

    println!("a: {}, b: {}",test1.as_ref().a(), test1.as_ref().b());
    println!("a: {}, b: {}",test2.as_ref().a(), test2.as_ref().b());
}
```

有的函数要求与之配合使用的futures是`Unpin`。对于没有`Unpin`的`Future`或`Stream`，您首先必须使用`Box::pin`（用于创建`Pin<Box<T>>`）或`pin_utils::pin_mut!`宏（用于创建`Pin<&mut T>`）来固定该值。 `Pin<Box<Fut>>`和`Pin<&mut Fut>`都可以作为futures使用，并且都实现了`Unpin`。

例如：

```rust,edition2018,ignore
use pin_utils::pin_mut; // `pin_utils` is a handy crate available on crates.io

// A function which takes a `Future` that implements `Unpin`.
fn execute_unpin_future(x: impl Future<Output = ()> + Unpin) { /* ... */ }

let fut = async { /* ... */ };
execute_unpin_future(fut); // Error: `fut` does not implement `Unpin` trait

// Pinning with `Box`:
let fut = async { /* ... */ };
let fut = Box::pin(fut);
execute_unpin_future(fut); // OK

// Pinning with `pin_mut!`:
let fut = async { /* ... */ };
pin_mut!(fut);
execute_unpin_future(fut); // OK
```

## Summary

1. 如果是`T:Unpin`（这是默认设置），则`Pin <'a, T>`完全等于`&'a mut T`。换句话说：`Unpin`表示即使固定了此类型也可以移动，因此`Pin`将对这种类型没有影响。
2. 如果是`T:!Unpin`，将`&mut T`变成固定的T需要unsafe。
3. 大多数标准库类型都实现了`Unpin`。对于您在Rust中遇到的大多数“常规”类型也是如此。由async / await生成的`Future`是此规则的例外。
4. 您可以在nightly使用功能标记添加`!Unpin`绑定到一个类型上，或者通过在stable将`std::marker::PhantomPinned`添加到您的类型上。
5. 您可以将数据固定到栈或堆上。
6. 将`!Unpin`对象固定到栈上需要`unsafe`。
7. 将`!Unpin`对象固定到堆并不需要`unsafe`。使用`Box::pin`可以执行此操作。
8. 对于`T:!Unpin`的固定数据，您必须保持其不可变，即从固定到调用drop为止，其内存都不会失效或重新利用。这是*pin*约束的重要组成部分。

["Executing `Future`s and Tasks"]: ../02_execution/01_chapter.zh.md
[the `Future` trait]: ../02_execution/02_future.zh.md

[ pin_utils ]:  https://docs.rs/pin-utils/