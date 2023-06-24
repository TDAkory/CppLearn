# Interfaces

An interface is a contract between two parts of a program. Precisely stating what is expected of a supplier of a service and a user of that service is essential. Having good (easy-to-understand, encouraging efficient use, not error-prone, supporting testing, etc.) interfaces is probably the most important single aspect of code organization.

## I.1: Make interfaces explicit


## I.2: Avoid non-const global variables
## I.3: Avoid singletons
## I.4: Make interfaces precisely and strongly typed
## I.5: State preconditions (if any)
## I.6: Prefer Expects() for expressing preconditions
## I.7: State postconditions
## I.8: Prefer Ensures() for expressing postconditions
## I.9: If an interface is a template, document its parameters using concepts
## I.10: Use exceptions to signal a failure to perform a required task
## I.11: Never transfer ownership by a raw pointer (T*) or reference (T&)
## I.12: Declare a pointer that must not be null as not_null
## I.13: Do not pass an array as a single pointer
## I.22: Avoid complex initialization of global objects
## I.23: Keep the number of function arguments low
## I.24: Avoid adjacent parameters that can be invoked by the same arguments in either order with ## different meaning
## I.25: Prefer empty abstract classes as interfaces to class hierarchies
## I.26: If you want a cross-compiler ABI, use a C-style subset
## I.27: For stable library ABI, consider the Pimpl idiom
## I.30: Encapsulate rule violations