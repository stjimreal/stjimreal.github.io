---
title: rust-memory-variables
date: 2021-10-19 14:48:45
tags: [rust lang, memory, syntax]
catagories: 
  - [rust lang, basics]
---

# Rust 基础：内存模型：1）所有权、Copy trait、借用规则和生命周期，2）智能指针、引用计数与内部可变性

## 内存模型

> 在冯诺依曼机的计算语言中，通常都是将内存分为静态区，栈上下文空间，堆上下文空间。在栈上下文中，通过 CPU 执行的函数体管理内存分配；在堆上下文中，通过运行时申请和回收内存空间。通常程序的内存泄漏都是发生运行时在处理堆内存过程中：C 语言将堆内存管理通过 malloc 接口完全交给程序员管理；C++ 通过智能指针和RAII机制部分地约束了堆内存的申请释放，但缺乏对线程安全的约束，引用的使用泛滥容易造成错误的并发处理；Java、C#主要采用可达性分析垃圾回收，Python、Objective-C 亦通过额外的运行时管理堆内存……

### Rust 内存管理与其他语言的区别

在 Rust 中，语言设计者发现，对于堆内存的需求存在两大方面：第一是希望当前上下文中可以有一段动态增长的堆空间，第二是希望在树或者图等非线性集合中通过多个指针引用一个内存、或者多线程中共享堆空间资源。在实际开发中，第一种需求占了大多数，因而可以将堆内存对象与栈空间绑定所有权，只能有一个变量拥有一个内存对象的所有权，如果发生了传参要么是所有权发生了转移，原有变量不再可用，要么是实现了拷贝，要么是在一定范围内将引用传递给了其他函数。而且这个引用在一段上下文中只能有一个是可变的，以确保线程安全。第二种情况在后面介绍。

## 静态约束：基于栈空间使用周期的堆空间动态生命周期(RAII 实现的静态检查)

### 所有权

在一段函数栈上下文中，一个值只有一个所有者。如果有修改只能通过**唯一所有者**修改或者通过唯一的可变借用修改，如果。

### Copy trait

> `Copy`、`Send`、`Sized`、`Sync`是`std::marker`中的标记 `trait`，无需任何实现代码，但是这些约定会影响编译器动态检查和代码生成。

实现了 Copy trait 的数据类型，在函数传参时可以按值执行拷贝传递，否则将转移值的所有权，意味着原有的变量不再在原有上下文使用(variable is moved)。如果不希望移动，则需要使用**引用类型**进行**封装**，再在函数传参时拷贝引用类型。

> 注意：引用类型并非如同在 C++ 中的 & 间接类型，而是类似 C++ 中的指针，可以是常引用也可以是**可变引用**，或者称为**借用**。

`Copy trait` 一般在有指针以及有自定义 Drop trait的数据类型中并不实现，例如 Vec, &mut 类型，因为浅拷贝指针会导致所有权滥用，所以一般使用其他的数据结构进行此类操作，我们可以认为，[Rust 中只有 POD(C++语言中的Plain Old Data) 类型才有资格实现 `Copy trait`](https://zhuanlan.zhihu.com/p/21730929)

### 借用规则

#### 可变借用与只读借用

1. 一个可变借用在上下文中与任意只读借用以及可变借用互斥
2. 只读借用不互斥

#### 因为借用是一个数据类型，因此借用有着自己本身的生命周期

### 生命周期

> 在函数中出现引用，则会需要编译器进行生命周期的判定。如果是在 类型定义或者 Trait 定义中，所有的引用生命周期默认与 self 一致；如果只有一个引用型输入，所有输出的生命周期与它一致；所有引用类型默认有独立的生命周期。

在函数上下文中，当引用的生命周期与实体的生命周期存在不确定的结果时，可以显式地表明引用关联的生命周期。

### 智能指针 Box/Rc/Arc/RefCell/Mutex/RwLock/(引用计数)(运行时检查)

## 动态约束

### Box 独占指针

`Box` 可以将任意类型在堆空间进行管理，但是 Box 指针管理是独占的，只能有一个引用。类似于 `unique_ptr`

### Rc 引用类似于强引用，共享所有权；Weak 为弱引用，通过 std::Rc::downgrade() 从 Rc 获取

Rc 引用计数，可以通过带引用计数的胖指针管理堆内存对象，在 `Rc::new()` 创建的源码中，实际上使用了 `Box::leak()` 获取了不受栈上下文控制的堆空间，因此可以在运行时动态的使用内存。然而 Rc 智能指针本身并不能修改值，可以通过内部可变引用修改值，需要用到 `RefCell`。

### 在多线程上下文，使用应用了原子引用计数的 Arc(Async Reference Count)，以及 Mutex/RwLock 封装内部可变引用

使用内部可变性，与外部可变性相比，外部可变性通过编译期检查发现变量更改不符合规则，内部可变性通过 panic 错误在运行时报告。

## 内存释放，调用 Drop trait

在 Drop trait 中不光可以进行堆内存的释放，也可以进行打开文件，以及其他 IO 资源的释放，由于 Rust 所有权唯一，所以可以无需考虑引用，简单地一次释放，即可完成最后的清理工作。