---
title: Rust-2
date: 2018-06-12 20:24:06
tags: Rust
categories: Tech
---

# Glitter的Rust之旅（2）
## 为什么我看这么慢，但是我还得继续看下去啊
### 继续Rust的编程环境的搭建
没错，装好VS2017之后我终于可以运行Rust程序了。
关于Atom的配置可以参考以下文章
> http://wiki.jikexueyuan.com/project/rust-primer/editors/atom.html

<!-- more -->
主要是装几个插件就OK
![](https://ws1.sinaimg.cn/large/8c185877gy1fs8n7jyma5j20qv0t3aba.jpg)
不过这里出了点问题，在Atom里只要一输入就会出现一大串报错，目前问题为啥会出现还不知道。。。下次。。在解决把。。。
![](https://ws1.sinaimg.cn/large/8c185877gy1fs8n9nqqgcj20on0faabu.jpg)
官方文档里面介绍了一个IDE，不过是Linux的，我也懒得去折腾linux所以就没有关注。

### Rust程序编译运行
这个和C/C++的gcc/g++编译运行挺像的，用过的应该不会陌生
编译：
```
rustc main.rs
```
编译完成之后就会生成可执行文件，直接运行即可（至于复杂的项目编译，以后再继续了解吧。。）

### Cargo是什么
Cargo是一个用于**构建系统和包的管理工具**，你可以用它来构建代码、下载依赖库以及编译这些库。emmmm，感觉很像SDK这样的存在？

### Cargo的使用
#### 使用Cargo快速构建项目
命令：
```
cargo new project_name --bin
```
![](https://ws1.sinaimg.cn/large/8c185877gy1fs8naax8w6j207h043743.jpg)
--bin不是必须的，这个参数是表示我们想创建一个可执行程序而不是一个库。执行完之后我们将会获得一个目录和两个文件，一个`Cargo.toml`和一个在`src`目录下的`main.rs`文件（并不止，还有其他文件啊，比如`Cargo.lock`用于跟踪项目的依赖，而且cargo还会贴心地帮你创建好git仓库，真是先进啊（棒读））。。

#### 通过Cargo编译运行
你可以只编译：
```
cargo build 
```
![](https://ws1.sinaimg.cn/large/8c185877gy1fs8n7k6tbkj20cv07y0sn.jpg)
执行build执行之后cargo会自动编译你的程序然后生成可执行文件，生成的目录在target的debug目录下，如果需要生成release版可以加上`--release`参数
如果希望一步到位，也可以：
```
cargo run
```
![](https://ws1.sinaimg.cn/large/8c185877gy1fs8n7kasykj20d602smwz.jpg)
那么可以直接编译运行一条龙
好了好了基本操作会了，终于可以继续学习Rust了
