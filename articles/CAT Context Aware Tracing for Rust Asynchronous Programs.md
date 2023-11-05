#### CAT: Context Aware Tracing for Rust Asynchronous Programs



##### 文章贡献：

1. context aware tracing methodology
2. a software framework based on the proposed methodology





##### context aware tracing methodology

###### Four context types

- Task context: task::spawn, task::exit
- Compiler-generated future context: async fn, async block
- User-implemented future context: impl Future
- Anonymous context: async closure, etc.



##### software framework

###### Dynamic Binary Instrumentation

use attribute to decorate the interested functions





