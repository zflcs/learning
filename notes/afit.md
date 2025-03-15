# AFIT

使用这种方式来手动达成 AFIT 的目的

```rust
use std::{future::Future, pin::Pin};
extern crate alloc;
fn main() {
    println!("Hello, world!");
    let mut context = std::task::Context::from_waker(std::task::Waker::noop());
    let mut filevec = Vec::<FileVtable>::new();
    filevec.push(TcpFile::new(1));
    filevec.push(TcpFile::new(23));
    filevec.push(EpollFile::new(2));
    filevec.push(EpollFile::new(3));
    for f in filevec {
        let fut = f.read();
        let mut fut = unsafe { Pin::new_unchecked(fut) };
        let _ = fut.as_mut().poll(&mut context);
    }
}

pub struct TcpFile {
    pub inner: i32,
}

impl TcpFile {
    pub fn new(inner: i32) -> FileVtable {
        let file = Box::new(TcpFile { inner });
        FileVtable::new(Box::into_raw(file) as *const (), |file| {
            Box::new(async move {
                let file = file as *const TcpFile;
                let file = unsafe { &*file };
                println!("Reading from TcpFile: {}", file.inner);
                10
            })
        })
    }
}

pub struct EpollFile {
    pub inner: i32,
}

impl EpollFile {
    pub fn new(inner: i32) -> FileVtable {
        let file = Box::new(EpollFile { inner });
        FileVtable::new(Box::into_raw(file) as *const (), |file| {
            Box::new(async move {
                let file = file as *const EpollFile;
                let file = unsafe { &*file };
                println!("Reading from EpollFile: {}", file.inner);
                10
            })
        })
    }
}

type BoxFut = Box<dyn Future<Output = usize>>;

pub struct FileVtable {
    pub inner: *const (), // 擦除具体的文件类型，后续的操作根据这个指针来进行
    read: fn(*const ()) -> BoxFut, // 相比于使用 async_trait 等已有的宏，这种方式只会在最初创建 File 的时候进行一次堆分配，与 StackFuture 类似，但是不会使用固定的空间，后续只需要把这个返回值需要的 Box 优化掉
}

impl FileVtable {
    pub fn new(inner: *const (), f: fn(*const ()) -> BoxFut) -> Self {
        Self { inner, read: f }
    }

    // pub async fn read(&self) -> usize {
    //     let fut = unsafe { Pin::new_unchecked((self.read)(self.inner)) };
    //     fut.await
    // }
    pub fn read(&self) -> BoxFut {
        (self.read)(self.inner)
    }
}

```
