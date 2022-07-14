# [Libcurl](https://curl.se/)

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

## 参考资料

> [curl](https://curl.se/)
>
> [libcurl](https://curl.se/libcurl/c/libcurl-tutorial.html)
>
> [libcurl进行http网络编程](https://www.cnblogs.com/moodlxs/archive/2012/10/15/2724318.html)
> 