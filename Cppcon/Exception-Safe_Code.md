# Exception-Safe code in C++

> [Exception-Safe Coding in C++ By Jon Kalb](http://www.exceptionsafecode.com/)
>
> [Part 1](https://www.youtube.com/watch?v=N9bR0ztmmEQ&t=45s) |
> [Part 2](https://www.youtube.com/watch?v=UiZfODgB-Oc)

## What's the Problem?

- Seperation of Error Detection from Error Handling

## Solutions without Exception

1.1 Error Flagging

- errno
- "GetRrror" Function

1.2 Problems with the Error Flagging Approach

- Errors can be ignored
  - Errors are ignored by default
- Ambiguity about which call failed
- Code is tedious to read and write

2.1 Return Code

- Almost every APU returns a code
- Usually int or long
- Known set of error/status values
- Error codes relayed up the call chain

2.2 Problems with the Return Code Approach

- Errors can be ignored
  - Are ignored by default
  - if a single call "break the chain" by not returning an error, errors cases are lost
- Code is tedious to read and write
- Exception based coding addresses both of these issues

