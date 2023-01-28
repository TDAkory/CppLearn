# commands and parameters of CMake

## `PROJECT_SOURCE_DIR`

This is the source directory of the last call to the `project()` command made in the current directory scope or one of its parents. Note, it is not affected by calls to `project()` made within a child directory scope (i.e. from within a call to `add_subdirectory()` from the current scope).

## `add_library`

```makefile
add_library(<name> [STATIC | SHARED | MODULE]
            [EXCLUDE_FROM_ALL]
            [<source>...])
```

Adds a library target called `<name>` to be built from the source files listed in the command invocation. The `<name>` corresponds to the logical target name and must be globally unique within a project. The actual file name of the library built is constructed based on conventions of the native platform (such as `lib<name>.a` or `<name>.lib`).

## `option()`

```makefile
option(<variable> "<help_text>" [value])
```

If no initial <value> is provided, boolean OFF is the default value. If <variable> is already set as a normal or cache variable, then the command does nothing
