---
title: Rust-5
date: 2018-06-22 11:20:45
tags: Rust
categories: Tech
---

# Glitter的Rust之旅（5）
## 引用&借用：用了我的给我还回来
上回说到，如果我们把一个堆分配的变量直接用作函数参数给传进去，那么变量的所有权会发生变化，以至于为了能够保证调用函数之后还能继续使用这个变量我们需要通过返回值来返还所有权，但是这么做太麻烦了，真要这样编程这门语言那还是go die吧。所以就需要能够访问一个变量，但是不改变他的所有权的方式，这就是**引用（reference）**要干的事。

<!-- more -->
## 引用的用法
首先我们看什么都不做的情况，这个就是之前所说的所有权变化，导致主函数中所有权丢失而无法访问的情况：
```
fn caculate_length(s:String) -> usize{
    return s.len();
}

fn main() {
    let s1 = String::from("hello");
    let len = caculate_length(s1);
    println!("len of \"{}\" is {}", s1, len);
}
```
运行之后就会告诉你：
![](https://ws1.sinaimg.cn/large/8c185877gy1fsjrs4f7nnj20oh06ijrd.jpg)
如果我们仍然想要在主函数里面访问s1怎么办呢？那就得把所有权拿回来：
```
fn caculate_length(s:String) -> (usize, String){
    return (s.len(), s);
}

fn main() {
    let s1 = String::from("hello");
    let (len, s1) = caculate_length(s1);
    println!("len of \"{}\" is {}", s1, len);
}
```
这里用了一个元组来返回多个值，以及在主函数中使用了一个let解构（前面的知识，我居然还没忘，ho~），但是怎样简化这个过程呢？
```
fn caculate_length(s:&String) -> usize{
    return s.len();
}

fn main() {
    let s1 = String::from("hello");
    let len = caculate_length(&s1);
    println!("len of \"{}\" is {}", s1, len);
}

```
这里参数定义以及调用的时候，都加了一个`&`，这就是Rust的引用标志（其实我觉得更像是C里面的指针），使用了引用就表明这个时候我们只是访问他的变量，但是所有权不会变化，就像指针一样，引用会指向s1的值，而所有权仍在s1（同样和指针类似，Rust存在解引用操作，用`*`实现，后面会讲）
![](https://kaisery.github.io/trpl-zh-cn/img/trpl04-05.svg)
这里将引用作为参数传给一个函数的操作叫做**借用（borrowing）**，意思就是你用了我的，用完之后还得还给我，他不属于你。

那么能不能修改一个引用呢？不能，当然也能

## 可变引用与不可变引用
直接创建一个引用是不能被修改的，比如：
```
let s = String::from("cannot be changed")
let rs = &s;
```
这里rs就不能被修改，你不能把他传入一个函数然后还想给他加一截在后面。
不过如果你实在想要修改一个引用，不改就会死那种，你也可以声明可变引用：
```
let rs2 = &mut s;
```
这样你就可以通过rs2来修改s的值，当前，前提是s是一个mut的变量

还有需要注意的问题，Rust限制了你可以声明的可变引用的数量，也就是在特定的作用域里面，**特定的数据只能有一个可变引用**（防止数据竞争）以及**不可变引用与可变引用不能同时存在**（防止不可变的引用读到的数据被修改），所以下面的代码会出现问题：
```
fn change_length(s:&mut String){
    s.push_str(", hey");
}
fn main() {
    let mut s1 = String::from("hello");
    let s2 = &mut s1;
    change_length(s2);
    println!("s1 is \"{}\"", s1);
    println!("s2 is \"{}\"", s2);
}
```
![](https://ws1.sinaimg.cn/large/8c185877gy1fsjsd591d8j20j1077gll.jpg)
这个问题我想了很久，为什么s1不能被访问，我猜测就是println!调用的时候就会通过s1的不可变引用来进行，而这里出现了可变引用与不可变引用同时存在的情况，所以编译器直接把你给毙了。

## 悬垂引用
悬垂引用是指一个引用指向的内存空间被释放了，类似空指针的概念，比如：
```
fn void_reference() -> &String{
    let s = String::from("hello");
    &s
}
```
这个函数返回了位于函数内局部变量的s的引用，但是s一离开作用域就会被释放，这个时候s的引用就会变成一个悬垂引用，好在Rust编译器看到这个之后就会直接把你给毙掉，避免你出错。

