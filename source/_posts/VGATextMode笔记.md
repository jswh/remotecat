---
title: VGA Text Mode笔记
date: 2019-03-29 02:35:03
tags:
---

本文是 [Writing an OS in Rust (Second Edition)](https://os.phil-opp.com/second-edition/)第三篇文章[VGA Text Mode](https://os.phil-opp.com/vga-text-mode/) 的阅读笔记，同时对文章中我觉得不太明白的一些内容做了一些扩展。

#### VGA Text Mode

**Memory-mapped I/O** (**MMIO**) 是 CPU 驱动 I/O 外设的一种方式。简单得说，就是外设的操作地址（我不知道这里该如何解释）映射到了系统内存的一个区域，CPU 通过直接读写内存，来与 I/O 外设通信。
<!--more-->
Text Mode 是显卡进行显示输出的一种实现方式。显卡将显示区域划分成一个二维的矩阵（类似 excel ）每个单元格对应一个文字字符数据。单个字符确切的像素数据由显卡控制。

VGA text mode，全称应该是 VGA-compatible text mode，是 Text Mode 的一种实现，用于兼容 VGA 的设备（或者说兼容 VGA 的设备，都有这种模式）。VGA text mode 规定，每个字符数据是个 2 字节的数据，前 8 位为字符的 askii 码，之后四位是文字颜色，再之后三位是背景颜色，最后一位控制是否闪烁。整个显示区域划分为 25 行 80 列（25行，每行80个字符）。 ![image](http://ww1.sinaimg.cn/large/0071ouepgy1g1gacwcic8j30yy03jjrc.jpg)

图片来自wikipedia，图里面是小端序，但我们编程一般默认大端序，所以是反的

CPU 通过 MMIO 的方式来驱动显卡的 VGA text mode。具体规则是在内存地址 0xB8000 处有一个25 * 80，单个元素 2 字节大小的二维数组（VGA text buffer），对应显示区域的 25 行 80 列的显示区域。每个数组元素就是一个单元格内的显示数据。程序只要指定读写该区域的数据，就能实现显示控制。

#### Volatile variable

Volatile 的原意是挥发性的，不稳定。而 Volatile variable 指的是那些可能通过多种途径进行修改的数据。比如上面的 VGA text buffer，如果在程序中使用那么指针对应的数据，除了程序本身会修改，其他的程序也会修改，显卡驱动（bios?）也会修改。

这样的数据会有一个问题，就是很多时候对于单一程序只会进行读取/写入这种操作中的一种，比如 VGA text buffer 读取很多时候是显卡的事，程序则只负责写入。这样的数据，编译器会认为是没有意义的，很可能会在优化的时候被去掉，反而出现错误。所以一些语言中会把这样的数据标记出来。比如c、c++、java都有关键字`volatile`。MMIO 是 `volatile` 最常用的场景之一。

rust中没有 `volatile` 关键字，但是有一个 volatile 包可以实现相关功能。Volatile 包只有一个 Struct 即`Volatile`，实现 `read/write` 两个方法。这两个方法本质上是 `instrinsics` 包中的相关函数的包装，用来指示编译器（LLVM）行为。

#### 几个attribute

##### allow

rust 编译器检查非常严格，比如 dead_code、unused_mut 之类的。通过 `allow`，可以让编译器忽略相关检查。完整的列表通过 `rustc -W help` 命令查看。

##### derive

rust 编译器可以非常智能提供一些基本 trait 的自动实现。通过 `derive` 可以指示 rust 编译器做这些事情。derive` 指令支持的 trait 有

- [`Eq`](https://doc.rust-lang.org/std/cmp/trait.Eq.html), [`PartialEq`](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html), [`Ord`](https://doc.rust-lang.org/std/cmp/trait.Ord.html), [`PartialOrd`](https://doc.rust-lang.org/std/cmp/trait.PartialOrd.html)
- [`Clone`](https://doc.rust-lang.org/std/clone/trait.Clone.html)
- [`Copy`](https://doc.rust-lang.org/core/marker/trait.Copy.html)
- [`Hash`](https://doc.rust-lang.org/std/hash/trait.Hash.html)
- [`Default`](https://doc.rust-lang.org/std/default/trait.Default.html)
- [`Debug`](https://doc.rust-lang.org/std/fmt/trait.Debug.html)

文章里面用到了 `Debug`, `Clone`, `Copy`, `PartialEq`, `Eq`这几个 trait。

##### repr

repr 指示了编译器之后内容的数据在内存中形式。文章中设计到了下面几种

- repr(transparant)。与struct中的第一个元素相同，一般用于只有单元素的struct，比如文中的`ColorCode`
- repr(C)。以C语言的方式存储。
- repr(RUST)。以rust语言的方式存储，这个文章中没有，和上面那个对应，也是默认的值。

更多的repr可用值可以看这个[文档](https://doc.rust-lang.org/nomicon/other-reprs.html)。

##### macro_export

类似pub的作用，不过作用对象是macro。

#### static和lazy static

在 rust中 的全局变量，即 static 类型，其他方面都与局部变量相同，只有生命周期不同。static 类型变量有个固定的`’static` 生命周期；数据值会在编译期初始化，并在整个程序运行周期中存在。与const 类型的差别在于，static类型支持mut关键字，就是可以在运行时更改其数据。

static 类型如果在一个包中，同样也要依靠pub关键字，使得可以在外部访问。

由于static类型可以通过表达式来初始化，而初始化又在编译期进行，所以很多数据类型不能用于static类型的初始化（就是那些编译器推测不了数据大小的类型）。一种解决方案是用 const function，但是文中的例子用const function也不能实现（具体的我也还不是特别理解是怎么回事）。文中通过 lazy static 的方式来处理这个问题。

lazy_static 这个包，提供了一个`lazy_static!`的macro。直接把 static 数据类型的初始化包裹其中就可以了。比如文中的

```rust
use lazy_static::lazy_static;

lazy_static! {
pub static ref WRITER: Writer = Writer {
column_position: 0,
color_code: ColorCode::new(Color::Yellow, Color::Black),
buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
};
}

```



#### mutex

关于并发问题根源可以看 [这个文章](https://www.cnblogs.com/fanzhidongyzby/p/3654855.html)，说得比较清楚。

VGA text buff 只是在内存地址0xb8000上一个二维数组数据，但是从逻辑上它是一个I/O 资源。对资源的使用涉及到竞争的问题，尤其是上述还用了static 类型来保存全局的 VGA text buff 数据。

一般的编程中都会引入 mutex (互斥锁）来保证共享资源的操作安全。mutex是一个抽象上的概念，其本质是一种二元锁机制，即一个程序流（这里不想用线程，因为线程是操作系统概念，到目前为止还没到线程）要想执行操作 A，必须先判断另一个数据 B（一般是个 bool）符合一定状态（数据A是否使用中的状态）。而数据B的变化（compareAndSwap）必须是原子操作，这样才能确保数据B的变化是是顺序的可预期的。

一般 Mutex 包（比如rust的std::sync::Mutex）是需要用到操作系统相关功能的，这样可以在没有办法获得资源的时候让出cpu，并在锁的状态改变时唤醒。但是我们在写就是操作系统，所以文章中引入了一个不依赖操作系统的 mutex，即 spin 这个包中的 mutex。

###### spinlock

所谓spinlock，简单理解就是用某种方式（比如while循环），使得当前程序流（操作系统一般就是线程）占满cpu，就避免了其他的的程序流来抢占cpu导致上下文切换进而发生资源竞争。说得通俗一点，就是原先cpu是谁着急谁用导致了资源竞争，spinlock使得我永远着急，所以等我用完你再用。

这里有一个用rust和spinlock来实现mutex的[文章](https://zhuanlan.zhihu.com/p/47501027)，可以用来帮助理解这些概念。spin 包中的 mutex 实现也比较简单，值得一看。

##### 界面刷新算法

界面的刷新基于一般的命令行界面规则，由下往上刷新，也就是在检测到当前行已经填满或者检测到换行符时，整体向上移动一行。对应到VGA text buff 上，由于数组顺序到界面的顺序为由上往下，从左往右的(`x[0][0]`是左上角第一个字符)，所以就是所有当前的字符，赋值给二级坐标减一的位置（`x[i][j-1] = x[i][j]`）。同时，最下面一行用空白填充。

界面刷新算法可以说是整个文章中最简单部分了。可以学习的更多还是对VGA text buff的抽象。具体实现在文章中很清楚，不再赘述。