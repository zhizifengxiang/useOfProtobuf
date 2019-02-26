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

key和value在编码阶段连接在一起，输出到字节流中。当message解码时，parser需要跳过无法识别的field——旧程序不会因为无法识别新加入的field而崩溃。因此，在wire-format-message中，每个key实际有两个值：用户设置的field-number，以及



# 4，More value types
#5 embedded message
#6 optional and repeated elements
#7 field order
