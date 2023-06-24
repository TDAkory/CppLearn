# Philosophy

> Philosophical rules are generally not mechanically checkable. However, individual rules reflecting these philosophical themes are. Without a philosophical basis, the more concrete/specific/checkable rules lack rationale.

## P.1: Express ideas directly in code

```cpp
class Date {
public:
    Month month() const;  // do
    int month();          // don't
    // ...
};
```

## P.2: Write In ISO Standard C++

## P.3: Express intent

```cpp
// Bad
gsl::index i = 0;
while (i < v.size()) {
    // bad The implementation detail of an index is exposed (so that it might be misused), and i outlives the scope of the loop.
}

// Better
for (const auto& x : v) { /* do something with the value of x */ }  
```

## P.4: Ideally, a program should be statically type safe

Unfortunately, that is not possible. Problem areas:
> always suggest an alternative

* unions --- use `variant` (C++17)
* casts --- minimize their use; template can help
* array decay --- use `span` (from the GSL)
* range errors --- use `span`
* narrowing conversions --- use `narrow` or `narrow_cast` (from the GSL)

## P.5: Prefer compile-time checking to run-time checking

Code clarity and performance. You don’t need to write error handlers for errors caught at compile time.

## P.6: What cannot be checked at compile time should be checkable at run time

## P.7: Catch run-time errors early

* Look at pointers and arrays: Do range-checking early and not repeatedly
* Look at conversions: Eliminate or mark narrowing conversions
* Look for unchecked values coming from input
* Look for structured data (objects of classes with invariants) being converted into strings

## P.8: Don’t leak any resources

* prefer RAII
* Look at pointers: Classify them into non-owners (the default) and owners. Where feasible, replace owners with standard-library resource handles (as in the example above). Alternatively, mark an owner as such using owner from the GSL.
* Look for naked new and delete
* Look for known resource allocating functions returning raw pointers (such as fopen, malloc, and strdup)

## P.9: Don’t waste time or space

Many more specific rules aim at the overall goals of simplicity and elimination of gratuitous waste.

## P.10: Prefer immutable data to mutable data

better optimization, and you can't have a data race on a constant.

* Look for “messy code” such as complex pointer manipulation and casting outside the implementation of abstractions.

## P.12: Use supporting tools as appropriate

* Static analysis tools
* Concurrency tools
* Testing tools

Be careful not to become dependent on over-elaborate or over-specialized tool chains. Those can make your otherwise portable code non-portable.

## P.13: Use support libraries as appropriate

* The ISO C++ Standard Library
* The Guidelines Support Library