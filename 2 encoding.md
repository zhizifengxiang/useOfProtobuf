# 1，A simple message
(关键词:binary-wire-format，)

下面我们有一个简单的message定义。我们将a设置为150，并序列化message到output-stream中。编码后，会看到3-byte的二进制：
>08 96 01

```
message Test1 {
  optional int32 a = 1;
}
```

# 2，Base 128 variant

首先需要理解“variant”。variant是一种使用若干字节序列化整数的技术。值比较小的数字，占用字节更少。

除了最后一个byte，variant中的每个byte都有most-significant-bit(msb)集合，其表示后续有更多的byte。每个字节的低7位存储以补码表示的数字，并将其分成两个部分，而least-significant部分在最开始。

比如，下面的数字值位1，由于只有一个byte，因此msb没有设置。

>0000 0001

再看下面这个数字的值位300。首先，我们将byte中的msb移除——提示是否到了数据的最后一个byte。可以看出，由于有多个byte，因此第一个byte的msb被设置了。

>1010 1100 0000 0010

>1010 1100 0000 0010    
-> 010 1100 000 0010

然后，调整每个7-bit之间的位置——按照least-significant方式将两组进行排放。

> 010 1100 000 0010    
-> 000 0010 010 1100    
-> 10 010 1100    
-> 256 + 32 + 8 + 4 = 300

# 3，Message structure
protobuf中的message由键值对组成，message的bianry-version就是用了赋给每个field的数字来作为key。field名称和类型只有在解码端（decoding-end），通过参考message定义（比如.proto文件），才能识别。

key和value在编码阶段连接在一起，输出到字节流中。当message解码时，parser需要跳过无法识别的field——旧程序不会因为无法识别新加入的field而崩溃。因此，在wire-format-message中，每个key实际有两个值：用户设置的field-number（在.proto文件中），以及wire-type，其提供了后续值得长度，以便进一步读取解析数据。在多数语言实现中，这个key为一个tag。

下面展示了wire-type。

|type|meaning|used for|
|--|--|--|
|0|variant|int32, int64, uint32, uint64, sint32, sint64, bool, enum|
|1|64-bit|fixed64, sfixed64, double|
|2|length-delimited|string, bytes, embedded message, packed repeated fields|
|3|start group| groups(deprecated)|
|4|end group|groups (deprecated)|
|5|32-bit|fixed32, sfixed32, float|

streamed-message中的每个key都是一个值为(field_number << 3) | wired_type的variant，即每个数字最后的3bit都存储了wire-type。

现在，重现看下本篇最开始的例子。如下的message被编码成3个字节。

```
message Test1 {
  optional int32 a = 1;
}
// 编码之后
08 96 01
```

第一个数字是08, 展开成二进制形式，并丢弃msb，则有如下形式。获得最后的3bit为wire-type = 0, 并右移3 bit，获得field-number=1。这样，我们就得到field-number, 后续就是variant。
> 000 1000

接着，使用之前学到的decoding知识，将后面的2 byte进行解码，值为150：

```
96 01 -> 1001 0110 0000 0001
      -> 000 0001 ++ 001 0110 (丢掉msb，并以7bit为一组交换顺序，拼接)
      -> 1001 0110
      -> 128 + 16 + 4 + 2 = 150
```

# 4，More value types
### 1，signed integers
如上所示，所有wire-type = 0的类型，都被编码成variant。但是，signed int（sint32, sint64）和 标准int（int32,int64）在encode负数时，有巨大差异。

如果使用int32和int64作为负数类型，则variant总会占用10 byte长度，并作为一个非常大的unsigned integer。若选择signed type, 得到的variant使用zigzag编码方式，其更高效。

zigzag编码将signed integer映射为unsigned integer，这样绝对值较小的数（如-1），生成的variant的也会很小。所以，-1会被编码为1，而1编码为2， -2编码为3，以此类推。具体见下面表格：

|signed original |encoded As|
|--|--|
|0 |0|
|-1|1|
|1|2|
|-2|3|
|2147483647|4294967294|
|-2147483648|4294967295|

所以，所有的数n，若为sint32，则：

>(n << 1) ^ (n >> 31)

若为sint64, 则：

> (n << 1) ^ (n >> 63)

上面第二次移位操作，为算数右移，所以，移位后补进来的bit，要不全是0（n为正数），要不全是1（n为负数）。

### 2，non-variant numbers
non-variant number类型只包含两种：double和fixed64, wire-type为1，告诉parser其需要读取64-bit大小的数据，wire-type=5（float和fixed32）的情形相同，其指出会读取32bit的数据。这两种情况都用小端字节序类存储。

### 3，string
wire-type=2表示后续variant的长度将会给出。

```
mesasge Test2 {
    optional string b = 2;
}
// 编码值为：
12 07 "74 65 73 74 69 6e 67"
```

上面引号括起来的数字，为UTF-8编码方式的字符串“testing”。

```
0x12 -> field-number = 2, type = 2
0x07 -> 表示后面跟有7个字符。
```

# 5，embedded message

下面利用之前定义的Test1 message，来对Test3进行定义。

```
message Test3 {
  optional Test1 c = 3;
}

// 编码
1a 03 "08 96 01"
```
注意到，引号部分的数值，与之前编码的Test1完全相同（08 96 01），且前面有数字“03”，表示三个字节。嵌套message的处理方式与string的处理方式一致。

# 6，optional and repeated elements


# 7，field order
