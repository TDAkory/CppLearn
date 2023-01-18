# Useful Template

> Some useful template func for meta programming

## type traits related

### `std::decay`

Applies lvalue-to-rvalue, array-to-pointer, and function-to-pointer implicit conversions to the type T, removes cv-qualifiers, and defines the resulting type as the member typedef type. Formally:

* If T names the type "array of U" or "reference to array of U", the member typedef type is U*.
* Otherwise, if T is a function type F or a reference thereto, the member typedef type is std::add_pointer<F>::type.
* Otherwise, the member typedef type is std::remove_cv<std::remove_reference<T>::type>::type.
These conversions model the type conversion applied to all function arguments when passed by value.

The behavior of a program that adds specializations for decay is undefined.

## utility

### `std::integer_sequence`

The class template std::integer_sequence represents a compile-time sequence of integers. When used as an argument to a function template, the parameter pack Ints can be deduced and used in pack expansion.

```cpp
// T: an integer type to use for the elements of the sequence
//...Ints: a non-type parameter pack representing the sequence
template< class T, T... Ints >
class integer_sequence;

/* Helper Templates */

template<std::size_t... Ints>
using index_sequence = std::integer_sequence<std::size_t, Ints...>;

template<class T, T N>
using make_integer_sequence = std::integer_sequence<T, /* a sequence 0, 1, 2, ..., N-1 */ >;

template<std::size_t N>
using make_index_sequence = std::make_integer_sequence<std::size_t, N>;

```

## tuple related

### `std::tuple_size<std::tuple>`

Provides access to the number of elements in a tuple as a compile-time constant expression.

```cpp
// since c++11
template< class... Types >
struct tuple_size< std::tuple<Types...> >
    : std::integral_constant<std::size_t, sizeof...(Types)> { };
// since c++17
template< class T >
inline constexpr std::size_t tuple_size_v = tuple_size<T>::value;
```

### `std::get(std::tuple)`

Extracts the Ith element from the tuple. I must be an integer value in [0, sizeof...(Types)).

```cpp
template< std::size_t I, class... Types >
typename std::tuple_element<I, tuple<Types...> >::type&
    get( tuple<Types...>& t ) noexcept;
```

Extracts the element of the tuple t whose type is T. Fails to compile unless the tuple has exactly one element of that type.

```cpp
template< class T, class... Types >
constexpr T& get( tuple<Types...>& t ) noexcept;
```