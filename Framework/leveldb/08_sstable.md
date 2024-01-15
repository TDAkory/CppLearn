# SSTable

> doc/table_format.md

SSTable是Leveldb的核心之一，是表数据最终在磁盘上的物理存储。也是体量比较大的模块。

```shell
    <beginning_of_file>
    [data block 1]
    [data block 2]
    ...
    [data block N]
    [meta block 1]
    ...
    [meta block K]
    [metaindex block]
    [index block]
    [Footer]        (fixed size; starts at file_size - sizeof(Footer))
    <end_of_file>
```