#1 define message type
首先定义一个简单的message结构，其包含如下三个字段：查询字符串、页号、每页中结果数量。下面是.proto文件中message定义：

```
syntax = "proto3"; // 使用proto3语法，文件第一行必须指定，不能使用空行，或者注释代替。

message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}
```
###1 assign field number
每个至少有三部分组成：类型（scaler-type, enum, message）、名称、数字。

其中，每个field的数字独一无二，其用于在message binary format中标识该字段，一旦使用某个数字，其他field就不能再用该数字。[1-15]表示使用1 byte来编码，

#2 scaler value type
#3 default values
#4 enumeration
#5 use other message type
#6 nested type
#7 update a message type
#8 unknow field
#9 Any
#10 Oneof
#11 Maps
#12 Packages
#13 define service
#14 json mapping
#15 options
#16 generate your class
