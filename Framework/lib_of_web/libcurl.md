# [Libcurl](https://curl.se/)

- [Libcurl](#libcurl)
  - [From Top Above](#from-top-above)
  - [基本用法](#基本用法)
  - [接口](#接口)
    - [全局初始化](#全局初始化)
    - [Http Headers used by Libcurl](#http-headers-used-by-libcurl)
  - [Topics](#topics)
    - [libcurl与连接池](#libcurl与连接池)
    - [`curl_socketpair`](#curl_socketpair)
  - [参考资料](#参考资料)


## From Top Above

- [libcurl API overview](https://curl.se/libcurl/c/libcurl.html)

从宏观的视角看，libcurl包含如下几类接口

* 初始化&清理&工具函数
* **easy interface**：an "easy handle" using `curl_easy_init` for a single individual transfer (in either direction). The easy interface is a synchronous interface with which you call curl_easy_perform and let it perform the transfer. When it is completed, the function returns and you can continue. 
* **multi interface**: The multi interface on the other hand is an asynchronous interface, that you call and that performs only a little piece of the transfer on each invoke.
* **share interface**: have multiple easy handles share certain data, even if they are used in different threads. This magic is setup using the share interface

## 基本用法

1. 调用`curl_global_init()`初始化libcurl
2. 调用`curl_easy_init()`函数得到 easy interface型指针
3. 调用`curl_easy_setopt()`设置传输选项
4. 根据`curl_easy_setopt()`设置的传输选项，实现回调函数以完成用户特定任务
5. 调用`curl_easy_perform()`函数完成传输任务
6. 调用`curl_easy_cleanup()`释放内存

## 接口

### 全局初始化

`CURLcode curl_global_init(long flags);`。

全局只调用一次。

如果这个函数在`curl_easy_perform`函数调用时还没调用，它讲由`libcurl`库自动调用，填充的`flags`参数是随机的。所以多线程下最好主动调用该函数以防止在线程中`curl_easy_init`时多次调用。

当`libcurl`库不再使用的时候，可以用`curl_global_cleanup`来清理所有资源。重复调用`curl_global_init`和`curl_global_cleanup`是不允许的，这两个接口在程序声明期内只应该被调用一次。

> CURL_GLOBAL_ALL              //初始化所有的可能的调用。
>
> CURL_GLOBAL_SSL              //初始化支持 安全套接字层。
>
> CURL_GLOBAL_WIN32            //初始化win32套接字库。When used on a Windows machine, it will make libcurl initialize the win32 socket stuff.
>
> CURL_GLOBAL_NOTHING          //没有额外的初始化。


### Http Headers used by Libcurl

使用`libcurl`来处理`HTTP`请求时, `libcurl`会自动设置一系列的`Http Header`参数。可以通过`CURLOPT_HTTPHEADER`选项来添加或删除`HTTP header`。

```cpp
CURLcode curl_easy_setopt(CURL *handle, CURLOPT_HTTPHEADER, struct curl_slist *headers);
```

```cpp
// Example
CURL *curl = curl_easy_init();
 
struct curl_slist *list = NULL;
 
if(curl) {
  curl_easy_setopt(curl, CURLOPT_URL, "https://example.com");
 
  list = curl_slist_append(list, "Shoesize: 10");
  list = curl_slist_append(list, "Accept:");
 
  curl_easy_setopt(curl, CURLOPT_HTTPHEADER, list);
 
  curl_easy_perform(curl);
 
  curl_slist_free_all(list); /* free the list again */
}
```

## Topics

### libcurl与连接池

Libcurl针对tcp连接采用了连接池管理，一次传输完成后，它将在“连接池”（有时也称为连接缓存）中保持N个连接处于活动状态，以便恰好能够重用现有连接之一的后续传输可以使用它而不是创建一个新连接。

重用一个连接而不是创建一个新的连接在速度和所需资源方面提供了显著的好处。

当libcurl准备建立一个新的连接来进行传输时，它首先会检查池中是否有可以重用的现有连接。**连接重用检查是在使用任何DNS或其他名称解析机制之前完成的，因此它完全基于主机名。**如果已经存在到正确主机名的实时连接，则还将检查许多其他属性（端口号，协议等），以确保可以使用它。

使用easy API，或更具体地说，使用curl_easy_perform()时，libcurl将使该池与特定的easy句柄关联。 然后重用同一简单句柄将确保它可以重用其连接。

使用multi API时，连接池将与multi句柄相关联。这允许您自由地清理和重新创建easy句柄，而不会有丢失连接池的风险，并且允许一个easy句柄使用的连接在以后的传输中被另一个简单句柄重用。只需重用multi句柄。

从libcurl 7.57.0开始，应用程序可以使用 share interface，以使其他独立的传输共享同一连接池。

### `curl_socketpair`

是 libcurl 内部使用的一个函数，主要用于创建**套接字对（socket pair）**。

**线程间通信**，当 libcurl 需要在线程间传递信号或数据时，会创建一对相互连接的套接字：

**唤醒事件循环**，在多路复用（multiplexing）场景中

```c
// 内部简化示例
curl_socket_t socks[2];
curl_socketpair(AF_UNIX, SOCK_STREAM, 0, socks);

// 将一个套接字加入多路复用监控
curl_multi_assign(multi_handle, socks[0], some_data);

// 需要唤醒时，向 socks[1] 写入数据
write(socks[1], "wake", 4);
```

**超时机制实现**，实现超时控制：

- 创建套接字对
- 设置超时时间
- 使用 `select()`/`poll()` 监控套接字
- 超时发生时写入数据触发

## 参考资料

- [curl](https://curl.se/)
- [curl://](https://everything.curl.dev/)
- [libcurl](https://curl.se/libcurl/c/libcurl-tutorial.html)
- [libcurl进行http网络编程](https://www.cnblogs.com/moodlxs/archive/2012/10/15/2724318.html)
