#### Redox 的思考

- Everything is URL
  - Everything is a scheme, identify by the URL

- URL 这个概念在微内核进程之间通信应该是高效的（这里似乎与 URL 后面接数据相关？？？），不用创建单独的数据结构，但是对于统一的抽象层面，将一个 URL 视作文件还是一个虚拟机，虽然可以用统一的 open 接口打开，但是这个更多的是取决于前面的 scheme


- 内核只提供最基本的 scheme，其余的 scheme 由其他的用户态的服务来提供


- resource 则像是内存中的一些缓冲区






- Everything is a URL 对于构建文件系统有什么借鉴之处？？？
  - 扁平化，目录用数据库来整理，表用标签来表示，对于文件的访问用 url 来实现