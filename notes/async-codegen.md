#### Implementation Details of async Rust

[Implementation Details of async Rust (swatinem.de)](https://swatinem.de/blog/async-codegen/)



它介绍了 core::task::future 中的 [GenFuture](https://github.com/rust-lang/rust/blob/4603ac31b0655793a82f110f544dc1c6abc57bb7/library/core/src/future/mod.rs#L64)

```rust
struct GenFuture<T: Generator<ResumeTy, Yield = ()>>(T);

impl<T: Generator<ResumeTy, Yield = ()>> Future for GenFuture<T> {
    type Output = T::Return;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        // SAFETY: Safe because we're !Unpin + !Drop, and this is just a field projection.
        let gen = unsafe { Pin::map_unchecked_mut(self, |s| &mut s.0) };

        // Resume the generator, turning the `&mut Context` into a `NonNull` raw pointer. The
        // `.await` lowering will safely cast that back to a `&mut Context`.
        match gen.resume(ResumeTy(NonNull::from(cx).cast::<Context<'static>>())) {
            GeneratorState::Yielded(()) => Poll::Pending,
            GeneratorState::Complete(x) => Poll::Ready(x),
        }
    }
}
```

上述代码片段中注释指出 .await 会将 Context<'static> 的原始指针转化成 &mut Context



async 函数被转化成 generator 的过程，在从 AST 转换成 HIR 时由编译器完成。  [make_async_expr](https://github.com/rust-lang/rust/blob/1286ee23e4e2dec8c1696d3d76c6b26d97bbcf82/compiler/rustc_ast_lowering/src/expr.rs#L566) 函数将 async { } 转化成 std::future::from_generator(<generator>)，而 [lower_expr_await](https://github.com/rust-lang/rust/blob/1286ee23e4e2dec8c1696d3d76c6b26d97bbcf82/compiler/rustc_ast_lowering/src/expr.rs#L665) 函数将 await 转化成一个循环，该循环将轮询底层的 future，并在 Pending 时产生结果，task_context = yield() 。

```rust
match ::std::future::IntoFuture::into_future(<expr>) {
	mut __awaitee => loop {
    	match unsafe { ::std::future::Future::poll(
        	<::std::pin::Pin>::new_unchecked(&mut __awaitee),
        	::std::future::get_context(task_context),
    	) } {
        	::std::task::Poll::Ready(result) => break result,
        	::std::task::Poll::Pending => {}
    	}
    	task_context = yield ();
	}
}
```



猜测：一旦 task_context = yield() 后，下一次 poll(&mut __awaitee, task_context) 时，根据 task_context 的状态，进行跳转，回到上一层的 poll 函数。

