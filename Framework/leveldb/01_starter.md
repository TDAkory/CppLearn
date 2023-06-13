# [LevelDB](https://github.com/google/leveldb.git)

LevelDB is a fast key-value storage library written at Google that provides an ordered mapping from string keys to string values.

## limitations

* This is not a SQL database. It does not have a relational data model, it does not support SQL queries, and it has no support for indexes.
* Only a single process (possibly multi-threaded) can access a particular database at a time.
* There is no client-server support builtin to the library. An application that needs such support will have to wrap their own server around the library.

## public header files

> Callers should not include or rely on the details of any other header files in this package. Those internal APIs may be changed without warning.

* **include/leveldb/db.h**: Main interface to the DB: Start here.

* **include/leveldb/options.h**: Control over the behavior of an entire database,
and also control over the behavior of individual reads and writes.

* **include/leveldb/comparator.h**: Abstraction for user-specified comparison function.
If you want just bytewise comparison of keys, you can use the default
comparator, but clients can write their own comparator implementations if they
want custom ordering (e.g. to handle different character encodings, etc.).

* **include/leveldb/iterator.h**: Interface for iterating over data. You can get
an iterator from a DB object.

* **include/leveldb/write_batch.h**: Interface for atomically applying multiple
updates to a database.

* **include/leveldb/slice.h**: A simple module for maintaining a pointer and a
length into some other byte array.

* **include/leveldb/status.h**: Status is returned from many of the public interfaces
and is used to report success and various kinds of errors.

* **include/leveldb/env.h**:
Abstraction of the OS environment.  A posix implementation of this interface is
in util/env_posix.cc.

* **include/leveldb/table.h, include/leveldb/table_builder.h**: Lower-level modules that most
clients probably won't use directly.
