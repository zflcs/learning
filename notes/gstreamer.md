# Gstreamer 学习笔记

## 基本概念熟悉

阅读博客园笔记[^1]即可。

## Mac 上编译

先安装好 meson 和 ninja 后，按照 github readme [^2] 中的提示设置好 builddir 且编译好后，使用 gst-env.py 创建出虚拟环境，这样 pkg-config 就可以找到 gstreamer 相关的依赖库。

## 创建 demo

使用 meson 进行编译，会省去很多麻烦，编译出来的产物可以直接运行。

## Gstreamer 基本数据结构

GList 双向链表

```c
typedef struct _GList GList;

struct _GList
{
  gpointer data;		// void*
  GList *next;
  GList *prev;
};
```

GSList 单向链表

```c
typedef struct _GSList GSList;

struct _GSList
{
  gpointer data;
  GSList *next;
};
```

GstParseContext 解析 pipeline 字符串格式时的上下文

```c
/* used by gstparse.c and grammar.y */
struct _GstParseContext {
  GList * missing_elements;
};
```

reference_t 存放着 pipeline 内部的 element 以及拥有的 pad 链表，具体含义尚且未知

```c
typedef struct {
  GstElement *element;
  gchar *name;
  GSList *pads;
} reference_t;
```

link_t 表示 pipeline 内部的 elements 之间的链接

```c
typedef struct {
  reference_t src;
  reference_t sink;
  GstCaps *caps;
  gboolean all_pads;
} link_t;
```

chain_t 表示 pipeline 中的一条链路

```c
typedef struct {
  GSList *elements; 	// elements 的单向链表
  reference_t first;	// 头 element
  reference_t last;		// 尾 element
} chain_t;
```

element_t

```c
typedef struct {
  gchar *factory_name;
  GSList *values;
  GSList *presets;
} element_t;
```

graph_t 记录了 pipeline 的链路以及 elements 的链路

```c
typedef struct _graph_t graph_t;
struct _graph_t {
  chain_t *chain; /* links are supposed to be done now */
  GSList *links;
  GError **error;
  GstParseContext *ctx; /* may be NULL */
  GstParseFlags flags;
};
```











## 参考链接

[^1]: [博客园笔记](https://www.cnblogs.com/xleng/p/10948838.html) 
[^2]:[GitHub README](https://github.com/GStreamer/gstreamer/blob/main/README.md)