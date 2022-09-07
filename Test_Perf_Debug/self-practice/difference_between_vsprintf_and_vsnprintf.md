# difference\_between\_vsprintf\_and\_vsnprintf

## 结论

vsprintf不会对字符进行截断，可能越界，造成运行时错误；

vsnprintf会对字符进行截断，并返回字符的实际长度；其存储上限是bufSize - 1；

## vsprintf

```cpp
#include <stdio.h>
#include <iostream>

using namespace std;

constexpr uint32_t DEFAULT_BUF_SIZE = 16;
char buf[DEFAULT_BUF_SIZE];

void PrintTest(const char *fmt, ...){
    va_list(args);
    va_start(args, fmt);
    uint32_t msgLen = vsprintf(buf, fmt, args);
    va_end(args);

    cout << "Len : " << msgLen << endl;
}

int main(){
    PrintTest("Hello World, xxxxxxxx");
    cout << buf << endl;
    return 0;
}
```

Output:

```text
/Users/admin/WorkPlace/CodeSnippet/cmake-build-debug/CodeSnippet
Len : 21
Hello World, xxxxxxxx

Process finished with exit code 0
```

## vsnprintf

```cpp
#include <stdio.h>
#include <iostream>

using namespace std;

constexpr uint32_t DEFAULT_BUF_SIZE = 16;
char buf[DEFAULT_BUF_SIZE];

void PrintTest(const char *fmt, ...){
    va_list(args);
    va_start(args, fmt);
    uint32_t msgLen = vsnprintf(buf, DEFAULT_BUF_SIZE, fmt, args);
    va_end(args);

    cout << "Len : " << msgLen << endl;
}

int main(){
    PrintTest("Hello World, xxxxxxxx");
    cout << buf << endl;
    return 0;
}
```

Output:

```text
/Users/admin/WorkPlace/CodeSnippet/cmake-build-debug/CodeSnippet
Len : 21
Hello World, xx

Process finished with exit code 0
```
