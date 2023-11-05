#### The Registers of Rust

[The registers of Rust - Without boats, dreams dry up](https://without.boats/blog/the-registers-of-rust/)



- **Core register:** defining the reified interface by hand with complete control and minimal compiler or library defined abstractions.
- **Consuming register:** actually consuming the reified type of the effect, closing the monad and finishing the effect.
- **Combinatoric register:** using library-defined combinators on the interface to operate over the reification with closures.
- **Control-flow register:** using compiler-expanded syntactic sugar to turn an ordinary looking function into one which constructs the effect-reified type.

|                           | Asynchrony  | Iteration        | Fallibility               |
| ------------------------- | ----------- | ---------------- | ------------------------- |
| **Reification**           | *Future*    | *Iterator*       | *Result/Option*           |
| **Core register**         | impl Future | impl Iterator    | Ok/Err and Some/None      |
| **Consuming register**    |             | for loop/collect | match/unwrap              |
| **Combinatoric register** |             | Iterator methods | Result and Option methods |
| **Control-flow register** | async/await |                  | ? operator                |



##### The missing control-flow register of iteration and fallibility

针对 Iteration，没有控制流变化的接口，只能半途而非。（可能理解不正确）

缺少带有 yield 语句的 generator。



##### Asynchronous iteration and AsyncIterator

异步函数通常伴随着 stream 接口，但是并没有提供 stream 接口的异步迭代器（作者认为实现 AsyncIterator，提供一个 async fn next 方法是走进了死胡同）。

这样会导致用户需要对 AsyncIterator 进行精确控制时，没有对应的 core register 可以操作，作为系统开发语言，这是不能接受的。作者提倡保留 poll_next 作为 asynchrony 和 iteration 之间的连接点。再结合带有 yield 的 generator，将会使得 async 与 sync 之间的更加自然。