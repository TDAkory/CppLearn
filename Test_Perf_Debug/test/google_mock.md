# [Google Mock](http://google.github.io/googletest/gmock_for_dummies.html)

- [gMock Cookbook](http://google.github.io/googletest/gmock_cook_book.html)
- [gMock for Dummiess](https://google.github.io/googletest/gmock_for_dummies.html)


It is easy to confuse the term fake objects with mock objects. Fakes and mocks actually mean very different things in the Test-Driven Development (TDD) community:

- **Fake** objects have working implementations, but usually take some shortcut (perhaps to make the operations less expensive), which makes them not suitable for production. An in-memory file system would be an example of a fake.
- **Mocks** are objects pre-programmed with expectations, which form a specification of the calls they are expected to receive.

**gMock** is a library (sometimes we also call it a “framework” to make it sound cool) for creating mock classes and using them. It does to C++ what jMock/EasyMock does to Java (well, more or less).

When using gMock,

1. first, you use some simple macros to describe the interface you want to mock, and they will expand to the implementation of your mock class;
2. next, you create some mock objects and specify its expectations and behavior using an intuitive syntax;
3. then you exercise code that uses the mock objects. gMock will catch any violation to the expectations as soon as it arises.

