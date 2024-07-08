# 基础
FFI是"Foreign Function Interface"的缩写，意为外部函数接口。
在编程中，FFI是一种机制，用于在不同的编程语言之间进行交互。
Rust的FFI允许Rust代码调用其他编程语言的函数，也允许其他编程语言调用Rust的函数。
通过FFI，我们可以实现Rust与其他编程语言之间的无缝集成，从而充分发挥各种编程语言的优势。
在Rust中，实现FFI的主要方式是使用extern关键字。
extern关键字用于声明外部函数，告诉Rust编译器这是一个外部函数，而不是Rust自己的函数。
FFI允许我们在Rust代码中调用其他编程语言的函数，同时也允许其他编程语言调用Rust代码。
# 调用C语言库函数
```rust
// 调用C语言库函数
extern "C" {
    fn printf(format: *const u8, ...) -> i32;
}

fn main() {
    let message = "Hello, World!";
    let format = b"%s\0";
    unsafe {
        printf(format.as_ptr(), message.as_ptr());
    }
}
```
# 底层系统交互
```rust
// 调用操作系统的API
extern "C" {
    fn Sleep(ms: u32);
}

fn main() {
    unsafe {
        Sleep(1000); // 等待1秒钟
    }
}
```
# 供外部调用
语言绑定和包装器
设置crate类型,编译生成dll库,可被外部调用
```xml
[lib]
name = "myfirst_rust_dll" #生成dll的文件名
crate-type = ["dylib"]
// Rust代码作为库供其他编程语言调用
#[no_mangle]
pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a + b
}
```
## 编译
```shell
cargo build
```
# 调用外部函数
```rust
// 调用外部函数
extern "C" {
    fn external_function(arg: i32) -> i32;
}

fn main() {
    let arg = 42;
    let result;
    unsafe {
        result = external_function(arg); // 调用外部函数
     }
    println!("Result: {}", result);
}
```
# 调用外部库
## 模板
```rust
#[link(name = "actual_lib_name_without_extname")] 
extern { 
    #[link_name = "actual_external_function_name"] // 支持对`FFI`函数重命名
    fn external_function_local_alias(_: *const c_char) -> *const c_char; // 原始指针`*const c_char`是对`FFI`安全的。
    ... 
}
```
### 解释
actual_lib_name_without_extname需要被替换为【链接库文件名（不含扩展名与
lib前缀）】
actual_external_function_name需要被替换为【外部函数真实名字】
external_function_local_alias需要被替换为【外部函数的本地别名】。即，根据本地命名规范，对外部函数·重命名。
## 案例
```rust
// 指定动态库文件名
#[link(name = "library_name")]
extern "C" {
    // 声明需要调用的函数签名
    fn function_name(arg1: i32, arg2: &str) -> i32;
}

fn main() {
    // 调用动态库中的函数
    let result = unsafe { function_name(1, "arg2") };
    println!("Result: {}", result);
}
```
# 库
## 动态库 dll
### 核心概念
```
动态库使用需要同时生成 *.dll 和 *.h
*.h 用于编译时给人看
*.dll 用于运行时使用
```
### 路径
link 宏查找路径
```
#[link(name = "library_name")]
```
### 方式一
将编译好的 *.dll 动态链接库放入 rustup 工具链库目录,例如目录
```
D:\Rust\RUSTUP\toolchains\stable-x86_64-pc-windows-gnu\lib\rustlib\x86_64-pc-windows-gnu\lib
```
## 静态库
#![no_std]
# cbindgen
## 概述
cbindgen 是一个从 Rust 库(这个库已面向暴露 C 接口进行设计)生成 C/C++ 头文件的工具
要在项目根目录创建 cbindgen.toml 文件
头文件只是在编译和代码编写阶段有效,具体还是要配合生成 动态链接库
## 安装
```shell
cargo install --force cbindgen
```
## 使用方式
以命令行工具的方式使用；
以库的方式在 build.rs 中使用；
### 命令行方式
对一个暴露了 C API 的 Rust crate,运行命令,会生成 my_header.h 头文件。
#### 指定配置文件
```shell
cbindgen --config cbindgen.toml --crate my_rust_library --output my_header.h
```
#### 不指定配置文件
```shell
cbindgen --crate my_rust_library --output my_header.h
```
#### 注意点
不是任意 Rust crate 都可以，而是已经做了暴露 C API 开发的库才行。因为 cbindgen 会去扫描整个源代码，把对接的接口抽出来；
可以通过 cbindgen.toml 这个配置文件，给 cbindgen 配置行为参数，参数很多，后面有参考链接。
### build.rs 方式
build.rs 的功能就是在编译期间，在编译真正的 crate 之前，先编译执行 build.rs 中的代码。
于是，可以在这里面做 on-fly 生成之类的工作。
存放在项目根目录,toml 文件的同级目录。
#### 文档
```
https://doc.rust-lang.org/cargo/reference/build-scripts.html
```
#### 模板
```rust
extern crate cbindgen;
 
use std::env;
 
fn main() {
    let crate_dir = env::var("CARGO_MANIFEST_DIR").unwrap();
 
    cbindgen::Builder::new()
      .with_crate(crate_dir)
      .generate()
      .expect("Unable to generate bindings")
      .write_to_file("bindings.h");
}
```
在 build.rs 方式里，也是可以配置 cbindgen.toml 参数的。
编译之后，生成的 bindings.h 就是我们要的 C 头文件。
#### 注意点
cbindgen 不但可以生成 C 头文件，还可以生成 C++ 的，甚至还可以生成 swift 可以用的头文件；
cbindgen 不是一包到底的，对 C 的支持相对成熟，对 C++ 的差一些。适当的时候，还是需要手动做很多事情的；
cbindgen 对 Rust 的泛型，有一定的支持；
#### cbindgen.toml 文件配置
cbindgen.toml 
```rust
https://rustcc.cn/article?id=5f347f7d-0268-4238-afc2-3cc0d3c55ce2
```
# 各语言互调
# 相关链接
https://rustcc.cn/article?id=4ca84a67-d972-4460-912e-a297ec5edc0a
https://www.bookstack.cn/read/RustPrimer/ffi-calling-ffi-function.md
https://blog.csdn.net/u012067469/article/details/105424363/