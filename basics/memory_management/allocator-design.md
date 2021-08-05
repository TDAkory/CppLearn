# allocator-design

## Design 1

```cpp
#include <cstddef>
#include <iostream>

using namespace std;

class Screen {
public:
    Screen(int x): i(x) {};
    int get() {return i;}

    void *operator new(size_t);
    void operator delete(void *, size_t);

private:
    Screen *next;
    static Screen *freeStroe;
    static const int screenChunk;

private:
    int i;
};

Screen *Screen::freeStore = 0;
const int Screen::screenChunk = 24;

void *Screen::operator new(size_t size)
{
    Screen *p;
    if(!freeStore) {
        size_t chunk = screenChunk * size;
        freeStore = p = reinterpre_cast<Screen *>(new char[chunk]);

        for (;p != &freeStore[ScreenChunk - 1]; p++) {
            p->next = p + 1;
        }
        p->next = 0;
    }
    p = freeStore;
    freeStore = freeStore->next;
    return p;
}

void Screen::operator delete(void *p, size_t size)
{
    (static_cast<Screen *>(p))->next = freeStore;
    freeStore = static_cast<Screen *>(p);
}
```

该设计多耗用了一个next的指针，malloc本身很快，这里减少malloc次数，效果没那么显著；

## Design 2

```cpp
//
// Created by zhaojieyi on 2021/4/11.
//

#include <cstddef>

class Airplane{
private:
    struct AirplaneRep {
        unsigned long miles;
        char type;
    };
private:
    union {
        AirplaneRep rep;
        Airplane *next;
    };
public:
    unsigned long getMiles(){return rep.miles;}
    char getType(){return rep.type;}
    void set(unsigned long m, char t){
        rep.miles = m;
        rep.type = t;
    }
public:
    static void *operator new(size_t size);
    static void operator delete(void *deadObject, int size);

private:
    static const int BLOCK_SIZE;
    static Airplane *headOfFreeList;
};

Airplane *Airplane::headOfFreeList;
const int Airplane::BLOCK_SIZE = 512;

void *Airplane::operator new(size_t size) {
    if(size != sizeof(Airplane))
        return ::operator new(size);

    Airplane *p = headOfFreeList;
    if (p) {
        headOfFreeList = p->next;
    } else {
        Airplane *newBlock = static_cast<Airplane *>(::operator new(BLOCK_SIZE * sizeof(Airplane)));
        for (int i = 1; i < BLOCK_SIZE; i++){
            newBlock[i].next = &newBlock[i+1];
        }
        newBlock[BLOCK_SIZE - 1].next = nullptr;
        p = newBlock;
        headOfFreeList = &newBlock[1];
    }
    return p;
}

void Airplane::operator delete(void *deadObject, int size) noexcept {
    if (deadObject == nullptr) return;
    if (size != sizeof(Airplane)) {
        ::operator delete(deadObject);
        return;
    }

    Airplane *carcass = static_cast<Airplane *>(deadObject);
    carcass->next = headOfFreeList;
    headOfFreeList = carcass;
}
```

上述两个方案，内存都没有还给操作系统，但也不能称之为内存泄漏，因为内存都在链表上；

## Design 3

上述两个设计，需要为每一个需要内存分配器的class都重载new和delete，不符合软件工程的设计思想，因此提取了公共的static allocator。

```cpp
//
// Created by zhaojieyi on 2021/4/11.
//

#include <cstddef>
#include <cstdlib>

class Allocator{
private:
    struct obj{
        struct obj * next;
    };
public:
    void *allocate(size_t);
    void deallocate(void *, size_t);
private:
    obj *freeStore = nullptr;
    const int CHUNK = 5;
};

void Allocator::deallocate(void *p, size_t) {
    ((obj*)p)->next = freeStore;
    freeStore= (obj *)p;
}

void *Allocator::allocate(size_t size) {
    obj *p;
    if (!freeStore){
        size_t chunk = CHUNK * size;
        freeStore = p = (obj *)malloc(chunk);

        for (int i = 0; i < CHUNK - 1; i++){
            p->next = (obj*)((char *)p + size);
            p = p->next;
        }
        p->next = nullptr;
    }
    p = freeStore;
    freeStore = freeStore->next;
    return p;
}
```

用法：

```cpp
class Foo{
public:
    long L;
    string str;
    static Allocator myAlloc;
public:
    Foo(long l) : L(l) {}
    static void *operator new(size_t size) {
        return myAlloc.allocate(size);
    }
    static void operator delete(void *pdead, size_t size){
        return myAlloc.deallocate(pdead, size);
    }
};

Allocator Foo::myAlloc;
```

## macro for static allocator

```cpp
#define DECLARE_POOL_ALLOC()     \
public:                          \
    void *operator new(size_t size) { return myAlloc.allocate(size); } \
    void operator delete(void *p) { myAlloc.deallocate(p, sizeof(p)); } \
protected:            \
    static Allocator myAlloc;

#define IMPLEMETN_POOL_ALLOC(class_name) \
Allocator class_name::myAlloc;
```

