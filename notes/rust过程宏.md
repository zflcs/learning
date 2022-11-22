### Rust 宏

目的：

- 生成其他代码
- 使用自定义构造扩展语言语法
- 帮助减少样板代码的数量

三种过程宏，都在函数上方打上标签

- [Function-like procedural macros](https://doc.rust-lang.org/reference/procedural-macros.html#function-like-procedural-macros)
  - 使用 `#[proc_macro]` ，在使用时，用函数名 + ！，例如 `foo!`，用于批量定义函数
- [Custom derive procedural macros](https://doc.rust-lang.org/reference/procedural-macros.html#derive-macros)
  - 使用 `#[proc_macro_derive(XXX)]` ，在定义数据结构和枚举类型时使用 `#[derive(xxx)]`，会在编译期间修改属性
- [Custom attributes](https://doc.rust-lang.org/reference/procedural-macros.html#attribute-macros)
  - 使用 `#[proc_macro_attribute]` ，会在编译期间修改属性