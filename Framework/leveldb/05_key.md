# Key

> K-V存储，先看看Key有哪些类型
> db/dbformat.h

```cpp
enum ValueType { kTypeDeletion = 0x0, kTypeValue = 0x1 };
```

`ValueType`在这里标识一个值是否存活

## ParsedInternalKey & InternalKey & user_key

```cpp
struct ParsedInternalKey {
  Slice user_key;           // data + size
  SequenceNumber sequence;  // uint64_t
  ValueType type;           // above
};
```

```cpp
class InternalKey {
 private:
  std::string rep_;
 public:
  InternalKey(const Slice& user_key, SequenceNumber s, ValueType t) {
    AppendInternalKey(&rep_, ParsedInternalKey(user_key, s, t));
  }
};

static uint64_t PackSequenceAndType(uint64_t seq, ValueType t) {
  assert(seq <= kMaxSequenceNumber);
  assert(t <= kValueTypeForSeek);
  return (seq << 8) | t;
}

void AppendInternalKey(std::string* result, const ParsedInternalKey& key) {
  result->append(key.user_key.data(), key.user_key.size());
  PutFixed64(result, PackSequenceAndType(key.sequence, key.type));
}
```

如上可见，用户构建一个`InternalKey`，传入的实际key值被存放在`rep_`的头部，然后会copy序列号和type在尾部。

```shell
| User key (string) | sequence number (7 bytes) | value type (1 byte) |
```

`ParsedInternalKey` 和 `InternalKey` 可以轻易的互相转换。

## LookupKey & Memtable Key

Memtable的查询接口传入的是LookupKey，它也是由User Key和Sequence Number组合而成的。

```cpp
LookupKey::LookupKey(const Slice& user_key, SequenceNumber s)

// | Size (int32变长)| User key (string) | sequence number (7 bytes) | value type (1 byte) |
```

LookupKey导出了三个函数，可以分别从LookupKey得到Internal Key，Memtable Key和User Key，如下：

```cpp
// Return a key suitable for lookup in a MemTable.
Slice memtable_key() const { return Slice(start_, end_ - start_); }

// Return an internal key (suitable for passing to an internal iterator)
Slice internal_key() const { return Slice(kstart_, end_ - kstart_); }

// Return the user key
Slice user_key() const { return Slice(kstart_, end_ - kstart_ - 8); }
```

其中**start_**是LookupKey字符串的开始，**end_**是结束，**kstart_**是user key字符串的起始地址。