# 0 简介
下面教程将从三个方面介绍protobuf的使用：
1. 在.proto文件中定义message
2. 使用protobuf的compiler
3. 使用C++ API来读写message

# 1 为什么使用protocol buffers?
现在我们需要创建一个通讯录，并对通讯录进行读写操作，里面记录了某人的name, ID, email, phone number。

下面有三种常用方式，来序列化和提取数据。
1. 以二进制形式直接存储。这种方式不可靠，需要考虑内存布局，大小端。可扩展性差。
2. 采用某种格式，将所有信息放到一个字符串中。不可靠，无法用于复杂数据。
3. XML。不好，消耗内存，处理速度慢。

使用流程：
1. 定义.proto文件，使用protobuf的compiler生成对应的class和method。
2. 使用生成的setter和getter来访问数据。


# 2 定义.proto文件

### 1 .proto文件内容
首先定义输入文件addressbook.proto，用于指导生成class和method。
```
syntax = "proto2";
package tutorial;

message Person {
  required string name = 1;
  required int32 id = 2;
  optional string email = 3;

  enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
  }

  message PhoneNumber {
    required string number = 1;
    optional PhoneType type = 2 [default = HOME];
  }

  repeated PhoneNumber phones = 4;
}

message AddressBook {
  repeated Person people = 1;
}
```

### 2. .proto文件说明

1. syntax关键字声明了protobuf使用的语法，为proto2——有proto3语法格式。
2. package关键字是为了防止命名冲突，其对应c++中的namespace。
3. message是.proto文件中存储数据的基本单元，每个message对应一个class。
4. 每个message中，field表示一条数据，用";"分隔。  
(1) 可以使用基本数据类型：bool, int32, float, double, string。也可以使用自定义message类型，如AddressBook使用Person。  
(2) 可以嵌套定义message，如Person中，嵌套定义了PhoneNumber。   
(3) 可以定义enum类型，如PhoneType。

5. 每个field使用required, optional, repeated三个关键字中的某个来进行修饰。具体意义为：   
(1)required: 必须初始化赋值。debug模式会检测此类型的field，是否已经赋值，若没赋值，则会assert failure。optimized模式不会进行检查，但parse未初始化的field总会失败（parse函数返回false）。其他行为和optional一致。   
(2) optiona: 不必初始化赋值。若没有初始化赋值，则程序会给field一个默认值，或者预先给出默认值(比如[default = HOME])。默认值通常为：numeric=0, string="(empty string)", bool=false。内嵌类型的message，默认值为“default instance”或者“prototype”，其不包含任何field。   
(3)repeated：表示该field会有多个实例，实例按照顺序保存，类似于动态分配数组。

注意：protobuf不支持继承——继承目的是方法复用，protobuf针对是数据读写，继承对protobuf没有意义。

# 3 编译.proto文件，生成对应的class和method
运行protoc程序，将.proto文件作为输入，会生成对应的header-file和source-file。

1. -I表示.proto文件所在目录——protobu无法找到对应的.proto目录。
2. --cpp_out表示生成C++源文件，并指出输出的.cc和.h文件的目录。
3. 最后给出输入文件: .proto文件。

下面指令生成两个文件：addressBook.pb.h和addressBook.pb.cc。

```
protoc -I=$src_dir --cpp_out=$dst_dir addressBook.proto
```

# 4 class和method
### 1 基本访问函数
下面是生成文件addressBook.pb.h中的部分内容。
1. getter的名称，同.proto文件中定义的field相同。如name对应的getter为"inline const std::string& name() const"。
2. setter的名称，是在getter前加前缀“set_”。如name对应的setter为“inline void set_name(const std::string& value)”。
3. 对于所有singular field（required和optional），会有前缀为“has_”的函数，若该field被赋值，则返回true。
4. 所有field都有对应前缀为"clear_"的函数，用于将变量清除为empty状态。

```
// name
  inline bool has_name() const;
  inline void clear_name();
  inline const ::std::string& name() const;
  inline void set_name(const ::std::string& value);
  inline void set_name(const char* value);
  inline ::std::string* mutable_name();

  // id
  inline bool has_id() const;
  inline void clear_id();
  inline int32_t id() const;
  inline void set_id(int32_t value);

  // email
  inline bool has_email() const;
  inline void clear_email();
  inline const ::std::string& email() const;
  inline void set_email(const ::std::string& value);
  inline void set_email(const char* value);
  inline ::std::string* mutable_email();

  // phones
  inline int phones_size() const;
  inline void clear_phones();
  inline const ::google::protobuf::RepeatedPtrField<::tutorial::Person_PhoneNumber >& phones() const;
  inline ::google::protobuf::RepeatedPtrField< ::tutorial::Person_PhoneNumber >* mutable_phones();
  inline const ::tutorial::Person_PhoneNumber& phones(int index) const;
  inline ::tutorial::Person_PhoneNumber* mutable_phones(int index);
  inline ::tutorial::Person_PhoneNumber* add_phones();
```

### 2 其他访问函数
对于比较复杂的类型，如string，自定义message类型，protobuf生成了前缀为“mutable_”的函数,返回指向该field的指针，我们可以利用指针，直接修改数据。未初始化的string存储一个空串。

同时，若field是一个singular-message(required或optional)，则只有"mutable_"函数，没有“set_”函数。

repeated-field提供了更多访问函数，比如Phone类型的field：
1. 使用_size函数查看实例数量。
2. 传入索引来获得指定数据。
3. 传入索引来修改指定数据。
4. 添加后续可修改的message实例——使用前缀为"add_"的函数。


# 5 enum 和 nested class
.proto文件中还定义了enum类型PhoneType，用户可以直接访问Person::PhoneType, Person::MOBILE等。

对于内嵌定义的class，可以直接以Person::PhoneNumber来访问。注意，PhoneNumber实际上的定义形式为Person_PhoneNumber，由于无法forward-declare嵌套class，此处使用了typedef来重新定义，从而方便声明。

# 6一些辅助方法
下面函数辅助操作message:
1. bool IsInitialized() const: 判断所有的required-field已经被赋值。
2. string DebugString() const: 返回一个人类能理解的字符串，对于调试很有用。
3. void CopyFrom(const Person& from): 用给定值覆盖当前的message.
4. void Clear(): 将所有的element清除为empty状态。


# 7 parse and serialize

下面方法让我们以bianry-format的形式访问message：

1. bool SerilizeToString(string* output) const : 序列化message，将生成的byte存储到string中。注意，存储的byte为binary-format，不是text-format。我们只是将string看做存放binary的容器。
2. bool ParseFromString(const string& data): 将给定string转换成一个message。
3. bool SerializeToOstream(ostream* output) const: 将message写入到输出流。
4. bool ParseFromIstream(istream* input): 从输入流中读取message。

# 8 write message
下面演示了，如何使用生成的类进行操作，首先创建对象，然后初始化数据，最后将数据写入输出流。程序首先从文件中读取一个AdressBook，根据用户输入，向其中添加一个Person，并将修改后的AddressBook重新写回到文件中。

```
#include <iostream>
#include <fstream>
#include <string>
#include "addressbook.pb.h"
using namespace std;

// 根据用户输入，给Person赋值
void PromptForAddress(tutorial::Person* person)
{
  int id;
  cout << "Enter person ID number: ";
  cin >> id;
  person->set_id(id);
  cin.ignore(256, '\n');

  cout << "Enter name: ";
  getline(cin, *person->mutable_name());

  string email;
  cout << "Enter email address (blank for none): ";
  getline(cin, email);
  if (!email.empty()) {
    person->set_email(email);
  }

  while (true) {
    string number;
    cout << "Enter a phone number (or leave blank to finish): ";
    getline(cin, number);
    if (number.empty()) {
      break;
    }

    tutorial::Person::PhoneNumber* phone_number = person->add_phones();
    phone_number->set_number(number);

    string type;
    cout << "Is this a mobile, home, or work phone? ";
    getline(cin, type);
    if (type == "mobile") {
      phone_number->set_type(tutorial::Person::MOBILE);
    } else if (type == "home") {
      phone_number->set_type(tutorial::Person::HOME);
    } else if (type == "work") {
      phone_number->set_type(tutorial::Person::WORK);
    } else {
      cout << "Unknown phone type.  Using default." << endl;
    }
  }
}

// Main function: 从文件中读取AddressBook，添加一个Person，然后写回数据
int main(int argc, char* argv[])
{

  // 检测被连接的库的版本与当前头文件的版本兼容
  GOOGLE_PROTOBUF_VERIFY_VERSION;

  if (argc != 2) {
    cerr << "Usage:  " << argv[0] << " ADDRESS_BOOK_FILE" << endl;
    return -1;
  }

  tutorial::AddressBook address_book;
  PromptForAddress(address_book.add_people()); // 添加Person

  // 写入磁盘文件
  fstream output(argv[1], ios::out | ios::trunc | ios::binary);
  if (!address_book.SerializeToOstream(&output)) {
    cerr << "Failed to write address book." << endl;
    return -1;
  }

  // Optional:  删除所有由libprotobuf分配的全局对象空间
  google::protobuf::ShutdownProtobufLibrary();

  return 0;
}
```
声明宏GOOGLE_PROTOBUF_VERIFY_VERSION，用来检测版本兼容性。其保证链接的库，和当前头文件版本一致。若不一致，则程序终止。.pb.cc文件会在startup是自动调用该宏。

main函数最后调用ShutdownProtobufLibrary()，会释放protobuf申请的所有空间。如果程序只执行一次，则不必调用该函数，若使用内存检测工具，或不断加载或卸载库，则需要调用该函数，防止内存泄漏。

# 9 read message
下面例子读取上面程序创建的数据，并打印出相应信息。
```
#include <iostream>
#include <fstream>
#include <string>
#include "addressbook.pb.h"
using namespace std;

// 迭代AddressBook中的所有Person，然后打印信息
void ListPeople(const tutorial::AddressBook& address_book)
{
  for (int i = 0; i < address_book.people_size(); i++) {
    const tutorial::Person& person = address_book.people(i);

    cout << "Person ID: " << person.id() << endl;
    cout << "  Name: " << person.name() << endl;
    if (person.has_email()) {
      cout << "  E-mail address: " << person.email() << endl;
    }

    for (int j = 0; j < person.phones_size(); j++) {
      const tutorial::Person::PhoneNumber& phone_number = person.phones(j);

      switch (phone_number.type()) {
        case tutorial::Person::MOBILE:
          cout << "  Mobile phone #: ";
          break;
        case tutorial::Person::HOME:
          cout << "  Home phone #: ";
          break;
        case tutorial::Person::WORK:
          cout << "  Work phone #: ";
          break;
      }
      cout << phone_number.number() << endl;
    }
  }
}

// 读取文件中的AddressBook，并打印出所有信息。
int main(int argc, char* argv[])
{
  // 检测版本兼容
  GOOGLE_PROTOBUF_VERIFY_VERSION;

  if (argc != 2) {
    cerr << "Usage:  " << argv[0] << " ADDRESS_BOOK_FILE" << endl;
    return -1;
  }

  tutorial::AddressBook address_book;

  fstream input(argv[1], ios::in | ios::binary);
  if (!address_book.ParseFromIstream(&input)) {
    cerr << "Failed to parse address book." << endl;
    return -1;
  }
  ListPeople(address_book);
  google::protobuf::ShutdownProtobufLibrary();

  return 0;
}
```



// 以下待看

# 10 extend a protocol buffer
如希望定义的数据类型向后兼容，或旧的数据类型向前兼容，需要遵循如下条款。在新的protobuf中：
1. 不能更改已经存在的field的tag
2. 不能添加、删除required类型的field
3. 可以删除optional或repeated类型的field
4. 可以添加optional或repeated类型的field，但是tag应该是新的（比如tag number为在当前protobuf中从没用过的，甚至已经删除的field也不能用过）。

遵循以上原则，可以让旧代码顺利读取修改后的message，新field将会被忽略。在旧代码中，删除的optional field被设置为默认值，删除的repeated field将为空。新代码将透明读取旧的消息。注意，新添加的optional field不会出现在旧的message中，所以需要使用函数has_来显式检测，或者在.proto文件中，提供一个默认值：[dfault = value]。若没有提供默认值，则使用程序给出的默认值：字符串为空串，bool为false，数字为0。另外，如果添加一个新的repeated field，新代码无法辨别其为空，还是从来没有设置值，因为没有has_函数提供。

# 11 optimization tips
protobuf库被高度优化，适当的使用也可以提高其性能。下面提供了一些建议：
1. 尽量复用message 对象。message会保存所有分配的内存以方面复用，甚至其已被回收掉。所以，如果需要处理大量同种类型的message，且后续还会用到，可以复用message对象。不过，如果message的尺寸不停变化，或者创建了一个比平常更大的message，message对象会变得十分臃肿。通过调用函数SpaceUsed来监测message对象大小，过大时候就要删掉。
2. 系统可能对于多线程分配大量小的对象，没有优化，可以试下Google的tcmalloc。

# 12 高级用法

除了简单的访问函数和序列化方法，protobuf还提供了更高级的功能。比如reflection，我们可以遍历message中的所有field，在无需指定任何message类型的情况下，直接操纵他们的值，该功能主要用于同其他编码方式进行转换：XML, json等。
reflection更高级的用法为比较同类型的两个message，或者开发一些“regular expression for protocol message”，我们可以写与某些内容匹配的表达式。
