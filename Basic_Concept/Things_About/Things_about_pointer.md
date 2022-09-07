# Things about Pointer

1. [When should I use raw pointers over smart pointers?](https://stackoverflow.com/questions/6675651/when-should-i-use-raw-pointers-over-smart-pointers)

    The rule would be this - if you know that an entity must take a certain kind of ownership of the object, always use smart pointers - the one that gives you the kind of ownership you need. If there is no notion of ownership, never use smart pointers.

2. 