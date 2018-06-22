---
title: Rust-5.5
date: 2018-06-22 16:40:17
tags: Rust
categories: Tech
---

# Glitter的Rust之旅（5.5）
## 另一个不获取所有权的引用——Slice
如果我们在一个函数里面希望通过操作能够返回一个字符串引用的一部分内容而不是全部，但是我们又没有所有权，不能对这个字符串做点什么，如何获取一个字符串的一部分引用，就是Slice要做的事。

<!-- more -->

## Slice的语法，从头到尾，切一段
获取一个字符串的一段内容，可以通过这样的方式：
```
fn main() {
    let s1 = String::from("hello world");
    let sl1 = &s1[0..5];
    let sl2 = &s1[6..11];
    println!("{}", sl1);
    println!("{}", sl2);
}
```
其中sl1和sl2就是两个Slice，Slice的语法就是`&name[start..end]`，`start`就是你要切取的这段内容的第一个字符的索引位置，`end`的话，有点绕，应该是你要切取这段内容的最后一个字符的下一个位置，比如上面的hello，是0~4索引的字符，声明Slice的时候是&s1[0..5]，另一个简单的算end的方法就是，用start加上你要截取的这段内容的长度，比如hello长度为5，start = 0， 那么end = start + 5 = 5（我真是个天才）
还有一个简化的点是，如果start为0，那么可以省略。比如
```
let sl1 = s1[0..5]
```
可以写成：
```
let sl1 = s1[..5]
```
同样，如果end是最后一个字节，那么end也可以省略，比如：
```
let sl2 = &s1[6..11];
```
可以写成：
```
let sl2 = &s1[6..];
```
那么如果我们start和end都不写会怎样呢，那就是创建了一个和原来内容一样的Slice啦。

## 前面用到的Slice
之前没注意，字符串的字面值其实就是一个全部引用的Slice，也就是：
```
let s = "hello world";
```
这里就是一个&str的Slice，所以s的内容是不能改变的

## 在函数中使用Slice
要在函数里面使用Slice我们得直到他是啥类型，当然Slice不是简单的&String类型，而是有个特别的类型，即`&str`
所以一个返回值为Slice的函数可以写成：
```
fn return_slice(s: &String) -> &str{

}
```
当然一个函数的参数也可以用到Slice：
```
fn SliceFunction(s: &str){

}
fn main(){
	let a = String::from("hello world");
    SliceFunction(a[..]);
}
```
这样做的好处是，当遇到上面字面值和一般字符串混用的情况，这样的代码能够更好地适应，因为字面值就是一个Slice，所以可以作为参数写进入，避免了一些不必要的错误。

## 其他类型也可以用Slice
其他的类型比如数组也可以使用Slice的语法，比如：
```
let a = [1,2.3,4];
let as = &a[0..2];
```
这里的as类型就是一个&[i32]

