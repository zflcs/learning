#### Rust 异步机制 Waker 和 Context 之间的关系



RawWaker 中两个数据结构：

```rust
pub struct RawWaker {
    data: *const (),		// 存放任意类型的数据结构的指针
    vtable: &'static RawWakerVTable,
}
```

Waker 内部的 wake 方法

```rust
    pub fn wake(self) {
        let wake = self.waker.vtable.wake;
        let data = self.waker.data;
        crate::mem::forget(self);
        unsafe { (wake)(data) };
    }
```

而 wake trait 的定义为

```rust
pub trait Wake {
    fn wake(self: Arc<Self>);

    fn wake_by_ref(self: &Arc<Self>) {
        self.clone().wake();
    }
}
```

因此，如果需要手动实现 waker，只需要在原始的 data 字段设置好目标数据结构的指针即可，在执行 wake 方法时，通过 vtable 找到对应的 waker 函数，而 data 即是 `Arc<self>`

embassy 里的做法是将 RawWaker 中的 data 设置成 TaskRef，它将 context 与中断关联起来，通过 context 可以找到 waker，waker 的 wake 方法可以对对应的数据结构进行处理，中间隐藏着一条连线

