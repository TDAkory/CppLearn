# crtstartup

windows下

CRT运行库启动函数分析： 在进入main函数之前，系统会调用CRT运行库的启动函数，做如下工作：

* 全局变量已完成初始化
* 堆的初始化
* I/O也完成了初始化
* Main调用

```c
******CRTStartUp()
{
  /*初始化一些操作系统版本的全局变量*/
  _osver   =   GetVersion();   

  _winminor   =   (_osver   >>   8)   &   0x00FF   ;   
  _winmajor   =   _osver   &   0x00FF   ;   
  _winver     =   (_winmajor   <<   8)   +   _winminor;   
  _osver      =   (_osver   >>   16)   &   0x00FFFF   ;   

  /*初始化堆*/
  if   (   !_heap_init(1)   )
      ……………..

/*初始化I/O ，这样在main函数中才能直接使用printf 之类的函数，使用windows的SHE机制*/
try {
  _ioinit();  

}__except (_XcptFilter(GetExceptionCode(),   GetExceptionInformation()) ){
    _exit(  GetExceptionCode() );
}

/*取得命令行参数*/
_wcmdln   =   (wchar_t   *)__crtGetCommandLineW();    
_wenvptr   =   (wchar_t   *)__crtGetEnvironmentStringsW();  

/*初始化main函数的argv参数*/ 
_wsetargv();   
/*初始化环境变量*/
_wsetenvp();   

/*初始化一些C数据,进行C库设置*/
_cinit();

/*调用main函数*/
mainret   =   main(__argc,   __argv,   _environ);   

/*等待main函数返回，然后退出进程*/
  exit(mainret);   
}
————————————————
版权声明：本文为CSDN博主「zacklin」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/zacklin/article/details/7630401
```

下面逐一分析各个阶段：

## 1. 初始化堆

调用的是\_heap\_init, 分析该函数：

```c
int __cdecl _heap_init (   int mtflag  /*多线程标志*/  )
{
        /*调用HeapCreate 创见进程堆，*/
        if ( (_crtheap = HeapCreate( mtflag ? 0 : HEAP_NO_SERIALIZE,
                                     BYTES_PER_PAGE, 0 )) == NULL )
            return 0;

        // Pick a heap, any heap
        __active_heap = __heap_select();

        return 1;
}
```

初始化堆是非常紧急的事情，否则其他的很多事情都做不了，如果堆初始化失败，那么进程就直接退出了。

## 3. I/O的初始化

首先I/O初始化函数需要在用户空间中建立stdin、stdout、stderr及其对应的FILE结构，使得程序进入main之后可以直接使用printf、scanf等函数。（其实printf和scanf操作的是FILE结构。）  
在linux中 stdin stdout stderr 的fd 分别为1，2，3 进程打开的文件fd从4开始。  
MSVC中FILE文件结构：  
struct \_iobuf {  
char \*\_ptr;  
int \_cnt;  
char \*\_base;  
int \_flag;  
int \_file; //通过该变量得到内部二维句柄表数组的两个下标，从而找到句柄  
int \_charbuf;  
int \_bufsiz;  
char \*\_tmpfname;  
};  
typedef struct \_iobuf FILE;

在MSVC的CRT中，已经打开的文件句柄的信息使用数据结构ioinfo来表示：  
typedef struct {  
intptr\_t osfhnd; //文件句柄  
char osfile;  
char pipech;  
} ioinfo;

在crt/src/ioinit.c中，有一个数组：  
int \_nhandle;  
ioinfo \* \_\_pioinfo\[64\]; // 等效于ioinfo \_\_pioinfo\[64\]\[32\];  
这就是每个进程用户态的打开文件表

通过FILE结构的 \_file 计算出二维数组下标，然后 取得 osfhnd 便可以得到句柄。

计算二维数组下标的宏（CRT内部使用）:  
\#define \_osfhnd\(i\) \( \_pioinfo\(i\)-&gt;osfhnd \)  
其中宏函数\_pioinfo的定义是：  
\#define \_pioinfo\(i\) \( \_\_pioinfo\[\(i\) &gt;&gt; 5\] + \(\(i\) & \(\(1 &lt;&lt; 5\) - 1\)\) \)  
FILE：\_file的第5位到第10位是第一维坐标（共6位），\_file的第0位到第4位是第二维坐标（共5位）。

MSVC的I/O初始化就是要构造这个二维的打开文件表。MSVC的I/O初始化函数\_ioinit定义于crt/src/ioinit.c中。首先，\_ioinit函数初始化了\_\_pioinfo数组的第一个二级数组：  
mainCRTStartup -&gt; \_ioinit\(\)：  
if \( \(pio = \_malloc\_crt\( 32 \* sizeof\(ioinfo\) \)\) //现在可以从对堆中分配内存了  
== NULL \)  
{  
return -1;  
}  
\_\_pioinfo\[0\] = pio;  
\_nhandle = 32;  
for \( ; pio &lt; \_\_pioinfo\[0\] + 32 ; pio++ \) {  
pio-&gt;osfile = 0;  
pio-&gt;osfhnd = \(intptr\_t\)INVALID\_HANDLE\_VALUE;  
pio-&gt;pipech = 10;  
}  
在这里\_ioinit初始化了的\_\_pioinfo\[0\]里的每一个元素为无效值，其中 INVALID\_ HANDLE\_VALUE是Windows句柄的无效值，值为-1。接下来，\_ioinit的工作是将一些预定义的打开文件给初始化，这包括两部分：  
\(1\) 从父进程继承的打开文件句柄，当一个进程调用API创建新进程的时候，可以选择继承自己的打开文件句柄，如果继承，子进程可以直接使用父进程的打开文件句柄。  
\(2\) 操作系统提供的标准输入输出。  
应用程序可以使用API GetStartupInfo来获取继承的打开文件，GetStartupInfo的参数如下：  
void GetStartupInfo\(STARTUPINFO\* lpStartupInfo\);  
STARTUPINFO是一个结构，调用GetStartupInfo之后，该结构就会被写入各种进程启动相关的数据。在该结构中，有两个保留字段为：  
typedef struct \_STARTUPINFO {  
……  
WORD cbReserved2;  
LPBYTE lpReserved2;  
……  
} STARTUPINFO;  
这两个字段的用途没有正式的文档说明，但实际是用来传递继承的打开文件句柄。当这两个字段的值都不为0时，说明父进程遗传了一些打开文件句柄。操作系统是如何使用这两个字段传递句柄的呢？首先lpReserved2字段实际是一个指针，指向一块内存，这块内存的结构如下：  
l 字节\[0,3\]：传递句柄的数量n。  
l 字节\[4, 3+n\]：每一个句柄的属性（各1字节，表明句柄的属性，同ioinfo结构的\_osfile字段）。  
l 字节\[4+n之后\]：每一个句柄的值（n个intptr\_t类型数据，同ioinfo结构的\_osfhnd字段）。  
\_ioinit函数使用如下代码获取各个句柄的数据：  
cfi\_len = \*\(\_\_unaligned int \*\)\(StartupInfo.lpReserved2\);  
posfile = \(char \*\)\(StartupInfo.lpReserved2\) + sizeof\( int \);  
posfhnd = \(\_\_unaligned intptr\_t \*\)\(posfile + cfi\_len\);  
其中\_\_unaligned关键字告诉编译器该指针可能指向一个没有进行数据对齐的地址，编译器会插入一些代码来避免发生数据未对齐而产生的错误。这段代码执行之后，lpReserved2指向的数据结构会被两个指针分别指向其中的两个数组，如图11-6所示。  
图11-6 句柄属性数组和句柄数组  
接下来\_ioinit就要将这些数据填入自己的打开文件表中。当然，首先要判断直接的打开文件表是否足以容纳所有的句柄：  
cfi\_len = \_\_min\( cfi\_len, 32 \* 64 \);  
然后要给打开文件表分配足够的空间以容纳所有的句柄：  
for \( i = 1 ; \_nhandle &lt; cfi\_len ; i++ \) {  
if \( \(pio = \_malloc\_crt\( 32 \* sizeof\(ioinfo\) \)\) == NULL \)  
{  
cfi\_len = \_nhandle;  
break;  
}  
\_\_pioinfo\[i\] = pio;  
\_nhandle += 32;  
for \( ; pio &lt; \_\_pioinfo\[i\] + 32 ; pio++ \) {  
pio-&gt;osfile = 0;  
pio-&gt;osfhnd = \(intptr\_t\)INVALID\_HANDLE\_VALUE;  
pio-&gt;pipech = 10;  
}  
}  
在这里，nhandle总是等于已经分配的元素数量，因此只需要每次分配一个第二维的数组，直到nhandle大于cfi\_len即可。由于\_\_pioinfo\[0\]已经预先分配了，因此直接从\_\_pioinfo\[1\]开始分配即可。分配了空间之后，将数据填入就很容易了：  
for \( fh = 0 ; fh &lt; cfi\_len ; fh++, posfile++, posfhnd++ \)  
{  
if \( \(\*posfhnd != \(intptr\_t\)INVALID\_HANDLE\_VALUE\) &&  
\(\*posfile & FOPEN\) &&  
\(\(\*posfile & FPIPE\) \|\|  
\(GetFileType\( \(HANDLE\)\*posfhnd \) !=  
FILE\_TYPE\_UNKNOWN\)\) \)  
{  
pio = \_pioinfo\( fh \);  
pio-&gt;osfhnd = \*posfhnd;  
pio-&gt;osfile = \*posfile;  
}  
}  
在这个循环中，fh从0开始递增，每次通过\_pioinfo宏来转换为打开文件表中连续的对应元素，而posfile和posfhnd则依次递增以遍历每一个句柄的数据。在复制的过程中，一些不符合条件的句柄会被过滤掉，例如无效的句柄，或者不属于打开文件及管道的句柄，或者未知类型的句柄。  
这段代码执行完成之后，继承来的句柄就全部复制完毕。接下来还须要初始化标准输入输出。当继承句柄的时候，有可能标准输入输出（fh=0,1,2）已经被继承了，因此在初始化前首先要先检验这一点，代码如下：  
for \( fh = 0 ; fh &lt; 3 ; fh++ \)  
{  
pio = \_\_pioinfo\[0\] + fh;  
if \( pio-&gt;osfhnd == \(intptr\_t\)INVALID\_HANDLE\_VALUE \)  
{  
pio-&gt;osfile = \(char\)\(FOPEN \| FTEXT\);  
if \( \(\(stdfh = \(intptr\_t\)GetStdHandle\( stdhndl\(fh\) \)\)  
!= \(intptr\_t\)INVALID\_HANDLE\_VALUE\)  
&& \(\(htype =GetFileType\( \(HANDLE\)stdfh \)\)  
!= FILE\_TYPE\_UNKNOWN\) \)  
{  
pio-&gt;osfhnd = stdfh;  
if \( \(htype & 0xFF\) == FILE\_TYPE\_CHAR \)  
pio-&gt;osfile \|= FDEV;  
else if \( \(htype & 0xFF\) == FILE\_TYPE\_PIPE \)  
pio-&gt;osfile \|= FPIPE;  
}  
else {  
pio-&gt;osfile \|= FDEV;  
}  
}  
else {  
pio-&gt;osfile \|= FTEXT;  
}  
}  
如果序号为0、1、2的句柄是无效的（没有继承自父进程），那么\_ioinit会使用GetStdHandle函数获取默认的标准输入输出句柄。此外，\_ioinit还会使用GetFileType来获取该默认句柄的类型，给\_osfile设置对应的值。  
在处理完标准数据输出的句柄之后，I/O初始化工作就完成了。我们可以看到，MSVC的I/O初始化主要进行了如下几个工作：  
l 建立打开文件表。  
l 如果能够继承自父进程，那么从父进程获取继承的句柄。  
l 初始化标准输入输出。  
在I/O初始化完成之后，所有的I/O函数就都可以自由使用了

