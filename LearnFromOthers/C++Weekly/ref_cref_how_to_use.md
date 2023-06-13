# [What Are std::ref and std::cref and When Should You Use Them?](https://www.youtube.com/watch?v=YxSg_Gzm-VQ)

## What is std::ref & std::cref

* the create of `std::reference_wrapper<Type>` and `std::reference_wrapper<const Type>`

```cpp
void func(int &i) {
    ++i;
}

struct Data{
    std::reference_wrapper<int> inref;
};

int main() {
    int local_i = 0;
    auto bound = std::bind(func, std::ref(local_i));
    bound();
    bound();
    return local_i;
}
```

* `std::bind` takes a copy of each variables
* if we want to pass by reference, we need to use `std::ref`

* Not recommend, by can work

```cpp
int main() {
    const std::array<int, 5> data{1,2,3,4,5};
    auto accumulator = [sum =0](int value) mutable {
        sum += value;
        return sum;
    }
    std::for_each(data.begin(). data.end(), std::ref(accumulator));
    return accumulator(0);
}
```
