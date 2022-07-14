# Things About [CRTP](https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern)

## What is

The curiously recurring template pattern (CRTP) is an idiom in C++ in which a class X derives from a class template instantiation using X itself as a template argument.[1] More generally it is known as F-bound polymorphism, and it is a form of F-bounded quantification.

## Form

```cpp
// The Curiously Recurring Template Pattern (CRTP)
template <class T>
class Base
{
    // methods within Base can use template to access members of Derived
};
class Derived : public Base<Derived>
{
    // ...
};
```

## Usage

### Static polymorphism

Typically, the base class template will take advantage of the fact that member function bodies (definitions) are not instantiated until long after their declarations, and will use members of the derived class within its own member functions, via the use of a cast.

```cpp
template <class T> 
struct Base
{
    void interface()
    {
        // ...
        static_cast<T*>(this)->implementation();
        // ...
    }

    static void static_func()
    {
        // ...
        T::static_sub_func();
        // ...
    }
};

struct Derived : Base<Derived>
{
    void implementation();
    static void static_sub_func();
};
```

In the above example, note in particular that the function Base<Derived>::implementation(), though declared before the existence of the struct Derived is known by the compiler (i.e., before Derived is declared), is not actually instantiated by the compiler until it is actually called by some later code which occurs after the declaration of Derived (not shown in the above example), so that at the time the function "implementation" is instantiated, the declaration of Derived::implementation() is known.

To elaborate on the above example, consider a base class with no virtual functions. Whenever the base class calls another member function, it will always call its own base class functions. When we derive a class from this base class, we inherit all the member variables and member functions that weren't overridden (no constructors or destructors). If the derived class calls an inherited function which then calls another member function, that function will never call any derived or overridden member functions in the derived class.

However, if base class member functions use CRTP for all member function calls, the overridden functions in the derived class will be selected at compile time. This effectively emulates the virtual function call system at compile time without the costs in size or function call overhead (VTBL structures, and method lookups, multiple-inheritance VTBL machinery) at the disadvantage of not being able to make this choice at runtime.

### Object Counter

```cpp
template <typename T>
struct counter
{
    static inline int objects_created = 0;
    static inline int objects_alive = 0;

    counter()
    {
        ++objects_created;
        ++objects_alive;
    }
    
    counter(const counter&)
    {
        ++objects_created;
        ++objects_alive;
    }
protected:
    ~counter() // objects should never be removed through pointers of this type
    {
        --objects_alive;
    }
};

class X : counter<X>
{
    // ...
};

class Y : counter<Y>
{
    // ...
};
```

### Polymorphic chaining

```cpp
// Base class
template <typename ConcretePrinter>
class Printer
{
public:
    Printer(ostream& pstream) : m_stream(pstream) {}
 
    template <typename T>
    ConcretePrinter& print(T&& t)
    {
        m_stream << t;
        return static_cast<ConcretePrinter&>(*this);
    }
 
    template <typename T>
    ConcretePrinter& println(T&& t)
    {
        m_stream << t << endl;
        return static_cast<ConcretePrinter&>(*this);
    }
private:
    ostream& m_stream;
};
 
// Derived class
class CoutPrinter : public Printer<CoutPrinter>
{
public:
    CoutPrinter() : Printer(cout) {}
 
    CoutPrinter& SetConsoleColor(Color c)
    {
        // ...
        return *this;
    }
};
 
// usage
CoutPrinter().print("Hello ").SetConsoleColor(Color.red).println("Printer!");
```

### Polymorphic copy construction

When using polymorphism, one sometimes needs to create copies of objects by the base class pointer. A commonly used idiom for this is adding a virtual clone function that is defined in every derived class. The CRTP can be used to avoid having to duplicate that function or other similar functions in every derived class.

```cpp
// Base class has a pure virtual function for cloning
class AbstractShape {
public:
    virtual ~AbstractShape () = default;
    virtual std::unique_ptr<AbstractShape> clone() const = 0;
};

// This CRTP class implements clone() for Derived
template <typename Derived>
class Shape : public AbstractShape {
public:
    std::unique_ptr<AbstractShape> clone() const override {
        return std::make_unique<Derived>(static_cast<Derived const&>(*this));
    }

protected:
   // We make clear Shape class needs to be inherited
   Shape() = default;
   Shape(const Shape&) = default;
   Shape(Shape&&) = default;
};

// Every derived class inherits from CRTP class instead of abstract class

class Square : public Shape<Square>{};

class Circle : public Shape<Circle>{};
```

## Pitfall

One issue with static polymorphism is that without using a general base class like AbstractShape from the above example, derived classes cannot be stored homogeneously – that is, putting different types derived from the same base class in the same container. For example, a container defined as std::vector<Shape*> does not work because Shape is not a class, but a template needing specialization. A container defined as std::vector<Shape<Circle>*> can only store Circles, not Squares. This is because each of the classes derived from the CRTP base class Shape is a unique type. A common solution to this problem is to inherit from a shared base class with a virtual destructor, like the AbstractShape example above, allowing for the creation of a std::vector<AbstractShape*>.