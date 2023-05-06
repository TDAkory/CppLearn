# [Ep 374 - C++23's out_ptr and inout_ptr](https://www.youtube.com/watch?v=DHKoN6ZBrkA)

A C++ 23 feature

```cpp

// in_out_ptr and out_ptr exist for compatibility between C++ smart pointers and C APIs

extern "C" {
    void get_data(int **ptr) {
        int * result = (int *)malloc(sizeof(int));
        *result = 42;

        *ptr = result;        
    }
}

int main() {
    std::unique_ptr<int, decltype([](int *ptr){ free(ptr);})> something;

    get_data(std::out_ptr(something));

    std::cout << *something << std::endl;
}
```

