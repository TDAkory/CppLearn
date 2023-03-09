# [Configure_file](https://cmake.org/cmake/help/latest/command/configure_file.html)

> Copy a file to another location and modify its contents.

```makefile
configure_file(<input> <output>
               [NO_SOURCE_PERMISSIONS | USE_SOURCE_PERMISSIONS |
                FILE_PERMISSIONS <permissions>...]
               [COPYONLY] [ESCAPE_QUOTES] [@ONLY]
               [NEWLINE_STYLE [UNIX|DOS|WIN32|LF|CRLF] ])
```

Copies an `<input>` file to an `<output>` file and substitutes variable values referenced as @VAR@ or ${VAR} in the input file content. Each variable reference will be replaced with the current value of the variable, or the empty string if the variable is not defined.

`#cmakedefine VAR ...` will be replaced with either `#define VAR ...` or `/* #undef VAR */` depending on whether VAR is set in CMake to any value not considered a false constant by the if() command. The "..." content on the line after the variable name, if any, is processed as above.
