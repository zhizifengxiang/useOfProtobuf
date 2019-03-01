# 1 define message type
首先定义一个简单的message结构，其包含如下三个字段：查询字符串、页号、每页中结果数量。下面是.proto文件中message定义。

1. 第一行syntax指定了proto文件的语法，如果没有指定，则默认为proto2格式。其不能为empty, 或者comment.
2. SearchRequest message定义3个field，每个field包含三个部分：类型，名字，field-number.

```
syntax = "proto3";

message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}
```

### 1 specify field type
上面所有的field类型都是scaler-type: integer和string。也可以指定复合类型，比如enumeration，或者其他message。

### 2 assign field number
message中的每个Field都有一个unique number——field number。这些数字用于在bianry format中唯一标识field，因此一旦使用message，数字不能变动。

数字范围1-15占用一个byte,这个byte中包含了field-number和field-type。范围在i6-2047占用2个byte。所以越是经常用到的数据，其field-number应该越小.允许的field-number范围时1-2^29-1（536，870，811).由于19000-19999被FieldDescriptor::kFirstReservedNumber through FieldDescriptor::kLastReservedNumber占用，因此这个范围的field-number被保留。

注意：
1. 已经使用过的field-number不能再用。
2. 19000-19999范围内的field-number不能使用。

### 3 specify field rule
message的field可以时下面两种：
1. singular：有0个或1个值，但是不能多于一个。
2. repeated：可以重复若干次，也可以为0次。repeated field出现顺序被保留下来。

### 4 add more message type
同一个.proto文件中可以定义多个message，比如可以定义答复message

```
message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}

message SearchResponse {
  ...
}
```

### 5 add comment
C++形式的comment都可以使用：/\*\*/和//

```
/* a multi-line comment */
message do {
  // do something
}
```

### 6 reserved field
若将某个field移除，或者comment掉，则未来user可以使用被释放的field-number。如果再用旧代码来load数据，就会引起严重错误（异常，privary bugs等）——field-number在二进制格式中，唯一标识某个field。防止错误的方式是，将这些field-number设置成reserved。如果未来的user尝试使用这些field-number，proto compiler将会报告出错。

```
message Foo{
  reserved 2, 15, 9 to 11;
  reserved "foo", "bar"; // 名称需要reserved, 会影响到json的序列化
}
```
注意：field-name和field-number不能掺和起来reserved。

### 7 what's generate from .proto
proto compiler将会根据使用的语言，来自动编译出对应的文件，生成代码提供：getter/setter, 向输出序列化，从输入parse等函数。

对于c++，会生成.h和.cc文件。

# 2 scaler value type
下表是proto中预定义的scaler类型。

||||
|--|--|--|
|
|
|
|
|
|
|
|
|

需要注意：
1. 赋值给field会进行有效性检查，保证是合法值。
2. 64bit和32bit integer在解码时，会被当作long，如果给出int，也会在被认为是int。
3. integer使用64bit机器码，string使用32bit机器码。

# 3 default values
输入过程中，parse encoded message时，若某个singular element没有值，则会为改field设置一个默认值。具体对应类型如下：

|||
|--|--|
|
|
|
|
|
|

repeated field的默认值为empty(通常为对应语言的empty list).需要注意，一旦scaler type message被parse，则无法确定改field是被设置成默认值，还是根本没有被设置。比如，如果一个bool的值为false时，会开启某些操作，而你又不想调用这些操作，那么就别设置它的默认值为false。同时，如果一个scaler message的被设置称其默认值，则该value不会被序列化。

# 4 enumeration
我们也可以定义枚举，比如在SearchRequest中，定义一个corpus枚举类型。

```
message SearchRequest {

  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;

  enum Corpus {
    UNIVERSAL = 0;
    WEB = 1;
    IMAGES = 2;
    LOCAL = 3;
    NEWS = 4;
    PRODUCTS = 5;
    VIDEO = 6;
  }
  Copus copus = 4;
}
```

需要注意，枚举类型的第一个值一定是0，因为：
1. 可以把0当作默认值。
2. 需要和proto2语法兼容——proto2中，第一个枚举变量的值总为默认值。

可以enum中的每个元素，定义number相同的元素，作为alias，此时需要这是allow_alias，若没指定，则compiler会报错。如下：

```
enum EnumAllowAlias {
  optional allow_alias = true;
  UNKNOW = 0;
  STARTED = 1;
  RUNNING = 1;
}
```

枚举类型必须在32-bit范围内，由于enum在传输过程中使用variant encoding，所以使用负数效率较低。enum可以定义在message内外部，若想引用定义在message内部的enum，则MessageType.EnumType.C++中会生成对应的enum类型定义。

反序列化过程中，无法识别的enum值会保存到message中，对于支持open enum type(enum element的实际值是对应语言中的基本值，比如c++中的enum实际上是整数)的语言，如C++和go, 其存储底层的实际整数值。对于close enum type,会提供特殊的accessor来传回对应的整数。无论怎样，这些无法识别的value都会被序列化。


# 5 use other message type
message可以使用定义在当前.proto文件中的其他message。比如：

```
message SearchRequest {
  repeated Result results = 1;
}

message Result {
  string url = 1;
  string title = 2;
  repeated string snippets = 3;
}
```

### 1 import definition
protobuf支持导入定义在其他proto文件中的message。像下面的import操作不具有传递性，other_protos.proto仅供当前文件import.proto使用。将来一些文件移动位置，则需要修改所有文件的import。所以，我们可以使用import public语句，并建立一个dummy(哑的) proto文件，来充当引用文件的桥梁。

```
// import.proto
import "myproject/other_protos.proto"

// new.proto
// all definition are moved here

// old.proto
// imported by clients
import public "new.proto"
import "other.proto"

// client.proto
import "old.proto"
// 使用ole.proto和new.proto，但是不能访问other.proto
```

compiler会搜索参数-I/ --proto_path后面的路径，来寻找import的proto文件，默认搜索compiler所在的文件。通常--proto_path指向的路径为整个project的根目录，所有的import应该写出完整路径。

### 2 use proto2 message type
proto2中的enum不能直接用在proto3的语法中，但是可以用在imported proto2 文件中。

# 6 nested type
如下定义了嵌套message。

```
message SearchRequest {
  message Result {
    string url = 1;
    string title = 2;
    repeated string snippets = 3;
  }
  repeated Result results = 1;
}
```

若希望在SearchRequest以外使用Result，则语法为：Parent.Type:

```
message SomeOtherMessage {
  SearchRequest.Result result = 1;
}
```
message可以任意深度进行嵌套：

```
message Outer {
  message Middle {
    message Inner {
      int64 val = 1;
      bool bval = 2;
    }
  }
}
```

# 7 update a message type



# 8 unknow field
# 9 Any
# 10 Oneof
### 1 use
### 2 feature
### 3 向后兼容
# 11 Maps
### 1 向后兼容
# 12 Packages
### 1 package and name resolution
# 13 define service
# 14 json mapping
### 1 json options
# 15 options
### 1 custom options
# 16 generate your class
