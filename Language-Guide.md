本指南介绍如何`protocol buffer` 语言构建`protocol buffer` 数据，包括`.proto`文件语法以及如何从`.proto`文件生成数据访问类。 它涵盖了protocol buffer语言的`proto3`版本：有关较早的`proto2`语法的信息，请参阅[`Proto2`语言指南](https://developers.google.com/protocol-buffers/docs/proto?hl=zh-CN)。

这是一个参考指南 - 对于使用本文档中描述的许多功能的分步示例，请参阅所选语言的[教程](https://developers.google.com/protocol-buffers/docs/tutorials?hl=zh-CN)（目前仅限`proto2`;更多`proto3`文档即将推出）。

##### 一、定义message类型
首先让我们看一个非常简单的例子。 假设您要定义搜索请求消息格式，其中每个搜索请求都有一个查询字符串，您感兴趣的特定结果页面以及每页的一些结果。 这是用于定义消息类型的`.proto`文件。
```
syntax = "proto3";

message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}
```
- 该文件的第一行指定您正在使用`proto3`语法：如果您不这样做，`protocol buffer`编译器将假定您正在使用`proto2`。 这必须是文件的第一个非空的非注释行。
- `SearchRequest`消息定义指定三个字段（名称/值对），每个字段对应要包含在此类消息中的每个数据。 每个字段都有一个名称和类型。

###### 1.1 指定字段类型
在上面的示例中，所有字段都是[标量类型](https://developers.google.com/protocol-buffers/docs/proto3?hl=zh-CN#scalar)：两个整数（`page_number`和`result_per_page`）和一个字符串（`query`）。 但是，您还可以为字段指定复合类型，包括[枚举](https://developers.google.com/protocol-buffers/docs/proto3?hl=zh-CN#enum)和其他消息类型。

###### 1.2 分配字段编号
如您所见，消息定义中的每个字段都有`唯一的编号`。 这些字段编号用于以[消息二进制格式](https://developers.google.com/protocol-buffers/docs/encoding?hl=zh-CN)标识字段，并且在使用消息类型后不应更改。 请注意，1到15范围内的字段编号需要一个字节进行编码，包括字段编号和字段类型（您可以在协[protocol buffer编码](https://developers.google.com/protocol-buffers/docs/encoding?hl=zh-CN#structure)中找到更多相关信息）。 16到2047范围内的字段编号占用两个字节。 因此，您应该为非常频繁出现的消息元素保留数字1到15。 请记住为将来可能添加的频繁出现的元素留出一些空间。

您可以指定的最小字段数为1，最大字段数为2的29次方 - 1或536,870,911。 您也不能使用数字19000到19999（`FieldDescriptor :: kFirstReservedNumber`到`FieldDescriptor :: kLastReservedNumber`），因为它们是为`protocol buffer`实现保留的 - 如果您在`.proto`中使用这些保留数字之一，`protocol buffer`编译器会抱怨。 同样，您不能使用任何以前[`保留`](https://developers.google.com/protocol-buffers/docs/proto3?hl=zh-CN#reserved)的字段编号。

###### 1.3 指定字段规则
消息字段可以是以下之一：
- 单数：格式正确的消息可以包含该字段中的零个或一个（但不超过一个）。
- `repeated`：该字段可以在格式良好的消息中重复任意次数（包括零）。 将保留重复值的顺序。

在`proto3`中，标量数字类型的重复字段默认使用压缩编码。
您可以在[Protocol Buffer Encoding](https://developers.google.com/protocol-buffers/docs/encoding?hl=zh-CN#packed)中找到有关`压缩编码`的更多信息。

###### 1.4 添加更多消息类型
可以在单个`.proto`文件中定义多种消息类型。 如果要定义多个相关消息，这非常有用 - 例如，如果要定义与`SearchResponse`消息类型对应的回复消息格式，则可以将其添加到相同的`.proto`：
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

###### 1.5 添加注释
要为`.proto`文件添加注释，请使用C/C++  `//`和`/*...*/`语法。
```
/* SearchRequest represents a search query, with pagination options to
 * indicate which results to include in the response. */

message SearchRequest {
  string query = 1;
  int32 page_number = 2;  // Which page number do we want?
  int32 result_per_page = 3;  // Number of results to return per page.
}
```

###### 1.6 保留字段
如果通过完全删除字段或将其注释来[更新](https://developers.google.com/protocol-buffers/docs/proto3?hl=zh-CN#updating)消息类型，则未来用户可以在对类型进行自己的更新时重用字段编号。 如果以后加载相同`.proto`的旧版本，这可能会导致严重问题，包括数据损坏，隐私错误等。 确保不会发生这种情况的一种方法是指定保留已删除字段的字段编号（和/或名称，这也可能导致JSON序列化问题）。 如果将来的任何用户尝试使用这些字段标识符，协议缓冲编译器将会抱怨。
```
message Foo {
  reserved 2, 15, 9 to 11;
  reserved "foo", "bar";
}
```
请注意，您不能在同一保留语句中混合字段名称和字段编号。

##### 1.7 从`.proto`产生什么？
当您在`.proto`上运行[`protocol buffer`编译器](https://developers.google.com/protocol-buffers/docs/proto3?hl=zh-CN#generating)时，编译器会使用您所选语言生成代码，您需要使用您在文件中描述的消息类型，包括获取和设置字段值，将消息序列化为输出流，并从输入流中解析您的消息。
- 对于`C++`，编译器会从每个`.proto`生成一个`.h`和`.cc`文件，并为您文件中描述的每种消息类型提供一个类。
- 对于`Java`，编译器生成一个`.java`文件，其中包含每种消息类型的类，以及用于创建消息类实例的特殊`Builder`类。
- `Python`有点不同 -  Python编译器生成一个模块，其中包含`.proto`中每种消息类型的静态描述符，然后与元类一起使用，以在运行时创建必要的`Python`数据访问类。
- 对于`Go`，编译器会生成一个`.pb.go`文件，其中包含文件中每种消息类型的类型。
- 对于`Ruby`，编译器生成一个带有包含消息类型的Ruby模块的`.rb`文件。
- 对于`Objective-C`，编译器从每个`.proto`生成一个`pbobjc.h`和`pbobjc.m`文件，其中包含文件中描述的每种消息类型的类。
- 对于`C＃`，编译器从每个`.proto`生成一个`.cs`文件，其中包含文件中描述的每种消息类型的类。
- 对于`Dart`，编译器会生成一个`.pb.dart`文件，其中包含文件中每种消息类型的类。

您可以按照所选语言的教程（即将推出的proto3版本）了解有关为每种语言使用API的更多信息。 有关更多API详细信息，请参阅相关[API参考](https://developers.google.com/protocol-buffers/docs/reference/overview?hl=zh-CN)（proto3版本即将推出）。

##### 二、标量值类型
标量消息字段可以具有以下类型之一。
`.proto` type | Notes | Go type | PHP type
---|---|---|---
double |  | float64 | float
float | | float32 | float
int32 | 使用可变长度编码。 编码负数的效率低 - 如果您的字段可能有负值，请改用sint32。 | int32 | integer
int64 | 使用可变长度编码。 编码负数的效率低 - 如果您的字段可能有负值，请改用sint64。 | int64 | integer/string
uint32 | 使用可变长度编码。 | uint32 | integer
uint64 | 使用可变长度编码。 | uint64 | integer/string
sint32 | 使用可变长度编码。 签名的int值。 这些比常规int32更有效地编码负数 | int32 | integer
sint64 | 使用可变长度编码。 签名的int值。 这些比常规int64更有效地编码负数 | int64 | integer/string
fixed32 | 总是四个字节。 如果值通常大于2的28次方，则比uint32更有效。| uint32 | integer
fixed64 | 总是八个字节。 如果值通常大于2的56次方，则比uint64更有效。|uint64 | integer/string
sfixed32 | 总是四个字节。 | int32 | integer
sfixed64 | 总是八个字节。 | int64 | integer/string
bool |  | bool | boolean
string | 字符串必须始终包含UTF-8编码或7位ASCII文本。 | string | string
bytes | 可以包含任意字节序列。 | []byte | string

该表显示`.proto`文件中指定的类型，以及自动生成的类中的相应[类型](https://developers.google.com/protocol-buffers/docs/proto3?hl=zh-CN#scalar)

##### 三、默认值
解析消息时，如果编码消息不包含特定的单数元素，则解析对象中的相应字段将设置为该字段的默认值。 这些默认值是特定于类型的：
- 对于strings，默认值为空字符串。
- 对于bytes，默认值为空字节。
- 对于bools，默认值为false。
- 对于数字类型，默认值为零。
- 对于[枚举](https://developers.google.com/protocol-buffers/docs/proto3?hl=zh-CN#enum)，默认值是第一个定义的枚举值，该值必须为0。
- 对于消息字段，未设置该字段。 它的确切值取决于语言。有关详细信息，请参阅生成的[代码指南](https://developers.google.com/protocol-buffers/docs/reference/overview?hl=zh-CN)。

重复字段的默认值为空（通常是相应语言的空列表）。

请注意，对于标量消息字段，一旦解析了消息，就无法确定字段是否显式设置为默认值（例如，布尔值是否设置为`false`）或者根本没有设置：您应该承担这一点 在定义消息类型时要注意。 例如，如果您不希望默认情况下也发生这种行为，那么当设置为false时，没有一个布尔值可以打开某些行为。 另请注意，如果标量消息字段设置为其默认值，则该值不会在线路上序列化。

有关默认值如何在生成的代码中工作的更多详细信息，请参阅所选语言的生成代码指南。

##### 四、枚举
在定义消息类型时，您可能希望其中一个字段只有一个预定义的值列表。 例如，假设您要为每`个SearchRequest`添加语料库字段，其中语料库可以是`UNIVERSAL`，`WEB`，`IMAGES`，`LOCAL`，`NEWS`，`PRODUCTS`或`VIDEO`。 您可以非常简单地通过向消息定义添加枚举，并为每个可能的值添加常量。

在下面的例子中，我们添加了一个名为`Corpus`的枚举，其中包含所有可能的值，以及一个类型为`Corpus`的字段：
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
  Corpus corpus = 4;
}
```
如您所见，`Corpus`枚举的第一个常量映射为零：每个枚举定义必须包含一个映射到零的常量作为其第一个元素。 这是因为：
- 必须有一个零值，以便我们可以使用0作为数字默认值。
- 零值必须是第一个元素，以便与`proto2`语义兼容，其中第一个枚举值始终是默认值。

您可以通过为不同的枚举常量指定相同的值来定义别名。 为此，您需要将`allow_alias`选项设置为`true`，否则协议编译器将在找到别名时生成错误消息。
```
enum EnumAllowingAlias {
  option allow_alias = true;
  UNKNOWN = 0;
  STARTED = 1;
  RUNNING = 1;
}
enum EnumNotAllowingAlias {
  UNKNOWN = 0;
  STARTED = 1;
  // RUNNING = 1;  // Uncommenting this line will cause a compile error inside Google and a warning message outside.
}
```
枚举器常量必须在32位整数范围内。 由于枚举值在线上使用[varint](https://developers.google.com/protocol-buffers/docs/encoding?hl=zh-CN)编码，因此负值效率低，因此不建议使用。 您可以在消息定义中定义枚举，如上例所示，也可以在外部定义枚举 - 这些枚举可以在`.proto`文件中的任何消息定义中重用。 您还可以使用语法`MessageType.EnumType`将一个消息中声明的枚举类型用作不同消息中字段的类型。

当您在使用枚举的`.proto`上运行`protocol buffer`编译器时，生成的代码将具有相应的Java或C ++枚举，这是Python的一个特殊的`EnumDescriptor`类，用于在运行时创建一组带有整数值的符号常量 生成的类。

在反序列化期间，无法识别的枚举值将保留在消息中，但是当反序列化消息时，如何表示这种值取决于语言。 在支持具有超出指定符号范围的值的开放枚举类型的语言中，例如C ++和Go，未知的枚举值仅作为其基础整数表示存储。 在具有封闭枚举类型（如Java）的语言中，枚举中的大小写用于表示无法识别的值，并且可以使用特殊访问器访问基础整数。 在任何一种情况下，如果消息被序列化，则仍然会使用消息序列化无法识别的值。

有关如何在应用程序中使用消息枚举的详细信息，请参阅所选语言的生成代码指南。

###### 4.1 保留值
如果通过完全删除枚举条目或将其注释掉来更新枚举类型，则未来用户可以在对类型进行自己的更新时重用该数值。 如果以后加载相同`.proto`的旧版本，这可能会导致严重问题，包括数据损坏，隐私错误等。 确保不会发生这种情况的一种方法是指定保留已删除条目的数值（和/或名称，这也可能导致JSON序列化问题）。 如果将来的任何用户尝试使用这些标识符，协议缓冲编译器将会抱怨。 您可以使用`max`关键字指定保留的数值范围达到最大可能值。
```
enum Foo {
  reserved 2, 15, 9 to 11, 40 to max;
  reserved "FOO", "BAR";
}
```
请注意，您不能在同一保留语句中混合字段名称和数值。

##### 五、使用其他消息类型
您可以使用其他消息类型作为字段类型。 例如，假设您希望在每`个SearchResponse`消息中包含`Result`消息 - 为此，您可以在同一`.proto`中定义`Result`消息类型，然后在`SearchResponse`中指定`Result`类型的字段：
```
essage SearchResponse {
  repeated Result results = 1;
}

message Result {
  string url = 1;
  string title = 2;
  repeated string snippets = 3;
}
```
###### 5.1 导入定义
在上面的示例中，`Result`消息类型在与`SearchResponse`相同的文件中定义 - 如果要用作字段类型的消息类型已在另一个`.proto`文件中定义，该怎么办？

您可以通过导入来使用其他`.proto`文件中的定义。 要导入另一个`.proto`的定义，请在文件顶部添加一个`import`语句：
```
import "myproject/other_protos.proto";
```
默认情况下，您只能使用直接导入的`.proto`文件中的定义。 但是，有时您可能需要将`.proto`文件移动到新位置。 现在，您可以在旧位置放置一个虚拟`.proto`文件，以使用`import public`概念将所有导入转发到新位置，而不是直接移动`.proto`文件并在一次更改中更新所有调用站点。 任何导入包含`import public`语句的`proto`的人都可以传递依赖导入公共依赖项。 例如：
```
// new.proto
// All definitions are moved here
```
```
// old.proto
// This is the proto that all clients are importing.
import public "new.proto";
import "other.proto"
```
```
// client.proto
import "old.proto";
// You use definitions from old.proto and new.proto, but not other.proto
```
协议编译器使用`-I/ -proto_path`标志在协议编译器命令行中指定的一组目录中搜索导入的文件。 如果没有给出标志，它将查找调用编译器的目录。 通常，您应将`--proto_path`标志设置为项目的根目录，并对所有导入使用完全限定名称。

###### 5.2 使用proto2的消息类型
可以导入`proto2`消息类型并在`proto3`消息中使用它们，反之亦然。 但是，`proto2`枚举不能直接用于`proto3`语法（如果导入的`proto2`消息使用它们就没关系）。

##### 六、嵌套类型
您可以在其他消息类型中定义和使用消息类型，如下例所示 - 此处`Result`消息在`SearchResponse`消息中定义：
```
message SearchResponse {
  message Result {
    string url = 1;
    string title = 2;
    repeated string snippets = 3;
  }
  repeated Result results = 1;
}
```
如果要在其父消息类型之外重用此消息类型，请将其称为`Parent.Type`：
```
message SomeOtherMessage {
  SearchResponse.Result result = 1;
}
```
您可以根据需要深入嵌套消息：
```
message Outer {                  // Level 0
  message MiddleAA {  // Level 1
    message Inner {   // Level 2
      int64 ival = 1;
      bool  booly = 2;
    }
  }
  message MiddleBB {  // Level 1
    message Inner {   // Level 2
      int32 ival = 1;
      bool  booly = 2;
    }
  }
}
```

##### 七、更新消息类型
如果现有的消息类型不再满足您的所有需求 - 例如，您希望消息格式具有额外的字段 - 但您仍然希望使用使用旧格式创建的代码，请不要担心！ 在不破坏任何现有代码的情况下更新消息类型非常简单。 请记住以下规则：
- 请勿更改任何现有字段的字段编号。
- 如果添加新字段，则使用“旧”消息格式按代码序列化的任何消息仍可由新生成的代码进行解析。您应该记住这些元素的默认值，以便新代码可以正确地与旧代码生成的消息进行交互。同样，您的新代码创建的消息可以由旧代码解析：旧的二进制文件在解析时只是忽略新字段。有关详细信息，请参阅“未知字段”部分
- 只要在更新的消息类型中不再使用字段编号，就可以删除字段。您可能希望重命名该字段，可能添加前缀“OBSOLETE_”，或者保留字段编号，以便`.proto`的未来用户不会意外地重复使用该编号。
- `int32`，`uint32`，`int64`，`uint64`和`bool`都是兼容的 - 这意味着您可以将字段从这些类型之一更改为另一种类型，而不会破坏向前或向后兼容性。如果从导线中解析出一个不适合相应类型的数字，您将获得与在C ++中将该数字转换为该类型相同的效果（例如，如果将64位数字作为`int32`读取，它将被截断为32位）。
- `sint32`和`sint64`彼此兼容，但与其他整数类型不兼容。
- 只要字节是有效的UTF-8，`string`和`bytes`是兼容的。
- 如果字节包含消息的编码版本，则嵌入消息与`bytes`兼容。
- `fixed32`与`sfixed32`兼容，`fixed64`与`sfixed64`兼容。
- `enum`与有线格式的`int32`，`uint32`，`int64`和`uint64`兼容（请注意，如果值不合适，将截断值）。但请注意，在反序列化消息时，客户端代码可能会以不同方式对待它们：例如，无法识别的`proto3`枚举类型将保留在消息中，但在反序列化消息时如何表示它是依赖于语言的。 `Int`字段总是保留它们的价值。
- 将单个值更改为新`oneof`的成员是安全且二进制兼容的。如果您确定没有代码一次设置多个字段，则将多个字段移动到新的字段可能是安全的。将任何字段移动到现有字段中是不安全的。

##### 八、未知字段
未知字段是格式良好的`protocol buffer`序列化数据，表示解析器无法识别的字段。 例如，当旧二进制文件解析具有新字段的新二进制文件发送的数据时，这些新字段将成为旧二进制文件中的未知字段。

最初，`proto3`消息在解析期间总是丢弃未知字段，但在3.5版本中，我们重新引入了未知字段的保存以匹配`proto2`行为。 在版本3.5及更高版本中，未知字段在解析期间保留并包含在序列化输出中。

##### 九、Any
`Any`消息类型允许您将消息用作嵌入类型，而无需使用`.proto`定义。 `Any`包含任意序列化消息作为字节，以及作为该消息类型的全局唯一标识符并解析的`URL`。 要使用任何类型，您需要导入`google/protobuf/any.proto`。
```
import "google/protobuf/any.proto";

message ErrorStatus {
  string message = 1;
  repeated google.protobuf.Any details = 2;
}
```
`type.googleapis.com/packagename.messagename`是给定消息类型的默认类型`URL`。

不同的语言实现将支持运行时库帮助程序以类型安全的方式打包和解压缩任何值 - 例如，在`Java`中，`Any`类型将具有特殊的`pack()`和`unpack()`访问器，而在C ++中则有`PackFrom()`和`UnpackTo()`方法：
```
/ Storing an arbitrary message type in Any.
NetworkErrorDetails details = ...;
ErrorStatus status;
status.add_details()->PackFrom(details);

// Reading an arbitrary message from Any.
ErrorStatus status = ...;
for (const Any& detail : status.details()) {
  if (detail.Is<NetworkErrorDetails>()) {
    NetworkErrorDetails network_error;
    detail.UnpackTo(&network_error);
    ... processing network_error ...
  }
}
```
目前，正在开发用于处理Any类型的运行时库。

如果您已熟悉`proto2`语法，则`Any`类型将替换[扩展](https://developers.google.com/protocol-buffers/docs/proto?hl=zh-CN#extensions)。

##### 十、Oneof
如果您有一个包含许多字段的消息，并且最多只能同时设置一个字段，则可以使用`oneof`功能强制执行此行为并节省内存。

除了一个共享内存中的所有字段之外，其中一个字段类似于常规字段，并且最多可以同时设置一个字段。 设置`oneof`的任何成员会自动清除所有其他成员。 您可以使用特殊`case()`或`WhichOneof()`方法检查`oneof`中的哪个值（如果有），具体取决于您选择的语言。

###### 10.1 使用Oneof
要在`.proto`中定义`oneof`，请使用`oneof`关键字，后跟您的`oneof`名称，在本例中为`test_oneof`：
```
message SampleMessage {
  oneof test_oneof {
    string name = 4;
    SubMessage sub_message = 9;
  }
}
```
然后，将`oneof`字段添加到`oneof`定义中。 您可以添加任何类型的字段，但不能使用重复的字段。

在生成的代码中，`oneof`字段与常规字段具有相同的`getter`和`setter`。 您还可以获得一种特殊方法来检查`oneof`中的值（如果有）。 您可以在相关API参考中找到有关所选语言的oneof API的更多信息。

###### 10.2 Oneof特征
- 设置`oneof`字段将自动清除`oneof`的所有其他成员。 因此，如果您设置多个字段，则只有您设置的最后一个字段仍具有值。
```
SampleMessage message;
message.set_name("name");
CHECK(message.has_name());
message.mutable_sub_message();   // Will clear name field.
CHECK(!message.has_name());
```
- 如果解析器在线路上遇到同一个`oneof`的多个成员，则在解析的消息中仅使用看到的最后一个成员。
- `oneof`不能`repeated`。
- Reflection API适用于其中一个字段。
- 如果您使用的是C ++，请确保您的代码不会导致内存崩溃。 以下示例代码将崩溃，因为已通过调用`set_name()`方法删除了`sub_message`。
```
SampleMessage message;
SubMessage* sub_message = message.mutable_sub_message();
message.set_name("name");      // Will delete sub_message
sub_message->set_...            // Crashes here
```
- 同样在C ++中，如果您使用`oneofs` `Swap()`两条消息，每条消息将以另一条消息结束：在下面的示例中，`msg1`将具有`sub_message`并且`msg2`将具有名称。
```
SampleMessage msg1;
msg1.set_name("name");
SampleMessage msg2;
msg2.mutable_sub_message();
msg1.swap(&msg2);
CHECK(msg1.has_sub_message());
CHECK(msg2.has_name());
```
###### 10.3 向后兼容性问题
添加或删除其中一个字段时要小心。 如果检查`oneof`的值返回`None/NOT_SET`，则可能意味着`oneof`尚未设置或已设置为`oneof`的不同版本中的字段。 没有办法区分，因为没有办法知道线上的未知字段是否是其中一个成员。

标签重用问题:
- 将字段移入或移出`oneof`：在序列化和解析消息后，您可能会丢失一些信息（某些字段将被清除）。 但是，您可以安全地将单个字段移动到新的字段中，并且如果已知只有一个字段被设置，则可以移动多个字段。
- 删除`oneof`字段并将其添加回：在序列化和解析消息后，这可能会清除当前设置的`oneof`字段。
- 拆分或合并`oneof`：这与移动常规字段有类似的问题。

##### 十一、Maps
如果要在数据定义中创建关联映射，`protocol buffer`提供了一种方便的快捷方式语法：
```
map<key_type, value_type> map_field = N;
```
...其中`key_type`可以是任何整数或字符串类型（因此，除了浮点类型和字节之外的任何标量类型）。 请注意，枚举不是有效的`key_type`。 `value_type`可以是除另一个`map`之外的任何类型。

因此，例如，如果要创建项目的`map`，其中每个Project消息都与字符串键相关联，则可以像这样定义它：
```
map<string, Project> projects = 3;
```
- `map`字段不能重复。
- `map`值的有线格式排序和地图迭代排序未定义，因此您不能依赖于特定顺序的`map`项目。
- 生成`.proto`的文本格式时，映射按键排序。 数字键按数字排序。
- 从线路解析或合并时，如果有重复的映射键，则使用最后看到的键。 - 从文本格式解析映射时，如果存在重复键，则解析可能会失败。
- 如果为映射字段提供键但没有值，则字段序列化时的行为取决于语言。 在C ++，Java和Python中，类型的默认值是序列化的，而在其他语言中没有任何序列化。

生成的`map`API目前可用于所有`proto3`支持的语言。 您可以在相关API参考中找到有关所选语言的地图API的更多信息。

###### 11.1 向后兼容性
`map`语法在线上等效于以下内容，因此不支持映射的`protocol buffer`区实现仍然可以处理您的数据：
```
message MapFieldEntry {
  key_type key = 1;
  value_type value = 2;
}

repeated MapFieldEntry map_field = N;
```
支持映射的任何`protocol buffer`实现都必须生成和接受上述定义可以接受的数据。

##### 十二、包
您可以向`.proto`文件添加可选的包说明符，以防止协议消息类型之间的名称冲突。
```
package foo.bar;
message Open { ... }
```
然后，您可以在定义消息类型的字段时使用包说明符：
```
message Foo {
  ...
  foo.bar.Open open = 1;
  ...
}
```
包说明符影响生成的代码的方式取决于您选择的语言：

- 在`C++`中，生成的类包含在`C++`命名空间中。例如，`Open`将位于命名空间`foo :: bar`中。
- 在Java中，除非在`.proto`文件中明确提供选项`java_package`，否则该包将用作Java包。
- 在Python中，将忽略package指令，因为Python模块是根据它们在文件系统中的位置进行组织的。
- 在Go中，除非在`.proto`文件中明确提供选项go_package，否则该包将用作Go包名称。
- 在Ruby中，生成的类包含在嵌套的Ruby命名空间中，转换为所需的Ruby大写形式（首字母大写;如果第一个字符不是字母，则PB_前置）。例如，`Open`将位于名称空间`Foo::Bar`中。
- 在C＃中，转换为`PascalCase`后，包将用作命名空间，除非您在`.proto`文件中明确提供选项`csharp_namespace`。例如，`Open`将位于名称空间`Foo.Bar`中。

###### 12.1 包和名称解析
`protocol buffer`语言中的类型名称解析与C++类似：首先搜索最里面的范围，然后搜索下一个范围，依此类推，每个包被认为是其父包的"内部"。 一个领先的'.' （例如，`.foo.bar.Baz`）意味着从最外层的范围开始。

`protocol buffer`编译器通过解析导入的`.proto`文件来解析所有类型名称。 每种语言的代码生成器都知道如何引用该语言中的每种类型，即使它具有不同的范围规则。

##### 十三、定义服务
如果要将消息类型与RPC（远程过程调用）系统一起使用，则可以在`.proto`文件中定义RPC服务接口，协议缓冲区编译器将使用您选择的语言生成服务接口代码和存根。 因此，例如，如果要使用获取`SearchRequest`并返回`SearchResponse`的方法定义RPC服务，可以在`.proto`文件中定义它，如下所示：
```
service SearchService {
  rpc Search (SearchRequest) returns (SearchResponse);
}
```
与协议缓冲区一起使用的最简单的RPC系统是[gRPC](https://grpc.io/)：一种在Google开发的语言和平台中立的开源RPC系统。 gRPC特别适用于protocol buffer，并允许您使用特殊的protocol buffer编译器插件直接从`.proto`文件生成相关的RPC代码。

如果您不想使用gRPC，也可以将`protocol buffer`与您自己的RPC实现一起使用。 您可以在`Proto2`语言指南中找到更多相关信息。

还有一些正在进行的第三方项目`为Protocol Buffers`开发RPC实现。 有关我们了解的项目的链接列表，请参阅[第三方加载项wiki页面](https://github.com/protocolbuffers/protobuf/blob/master/docs/third_party.md)。

##### 十四、JSON
`Proto3`支持JSON中的规范编码，使得在系统之间共享数据变得更加容易。 在下表中逐个类型地描述编码。

如果JSON编码数据中缺少某个值，或者其值为null，则在解析为`Protocol Buffer`时，它将被解释为相应的默认值。 如果字段`在Protocol Buffer`中具有默认值，则默认情况下将在JSON编码的数据中省略该字段以节省空间。 实现可以提供用于在JSON编码的输出中发出具有默认值的字段的选项。
###### 14.1 JSON选项
proto3 JSON实现可以提供以下选项：

- 使用默认值发出字段：默认情况下，`proto3` JSON输出中省略了具有默认值的字段。 实现可以提供覆盖此行为的选项，并使用其默认值输出字段。
- 忽略未知字段：默认情况下，`Proto3` JSON解析器应拒绝未知字段，但可以提供忽略解析中未知字段的选项。
- 使用`proto`字段名称而不是`lowerCamelCase`名称：默认情况下，`proto3` JSON打印机应将字段名称转换为`lowerCamelCase`并将其用作JSON名称。 实现可以提供使用proto字段名称作为JSON名称的选项。 `Proto3` JSON解析器需要接受转换后的`lowerCamelCase`名称和`proto`字段名称。
- 将枚举值作为整数而不是字符串发出：默认情况下，在JSON输出中使用枚举值的名称。 可以提供选项以使用枚举值的数值。

##### 十五、选项
`.proto`文件中的各个声明可以使用许多选项进行注释。 选项不会更改声明的整体含义，但可能会影响在特定上下文中处理它的方式。 可用选项的完整列表在`google/protobuf/descriptor.proto`中定义。

一些选项是文件级选项，这意味着它们应该在顶级范围内编写，而不是在任何消息，枚举或服务定义中。 一些选项是消息级选项，这意味着它们应该写在消息定义中。 一些选项是字段级选项，这意味着它们应该写在字段定义中。 选项也可以写在枚举类型，枚举值，服务类型和服务方法上; 但是，目前没有任何有用的选择。

以下是一些最常用的选项：
- `java_package`（文件选项）：要用于生成的Java类的包。 如果`.proto`文件中没有给出显式的`java_package`选项，那么默认情况下将使用`proto`包（使用`.proto`文件中的"package"关键字指定）。 但是，`proto`包通常不能生成好的Java包，因为预期`proto`包不会以反向域名开头。 如果不生成Java代码，则此选项无效。
```
option java_package = "com.example.foo";
```
- `java_multiple_files`（file option）：导致在包级别定义顶级消息，枚举和服务，而不是在`.proto`文件之后命名的外部类中。
```
option java_multiple_files = true;
```
- `java_outer_classname`（file option）：要生成的最外层Java类（以及文件名）的类名。 如果`.proto`文件中没有指定显式的`java_outer_classname`，则通过将`.proto`文件名转换为`camel-case`来构造类名（因此`foo_bar.proto`变为`FooBar.java`）。 如果不生成Java代码，则此选项无效。
```
option java_outer_classname = "Ponycopter";
```
- `optimize_for`（文件选项）：可以设置为`SPEED`，`CODE_SIZE`或`LITE_RUNTIME`。 这会以下列方式影响C++和Java代码生成器（可能还有第三方生成器）：
```
option optimize_for = CODE_SIZE;
```
- `cc_enable_arenas`（文件选项）：为C++生成的代码启用竞技场分配。
- `objc_class_prefix`（file option）：设置Objective-C类前缀，该前缀由此.proto提供给所有Objective-C生成的类和枚举。 没有默认值。 您应该使用Apple建议的3-5个大写字符之间的前缀。 请注意，Apple保留所有2个字母的前缀。
- `deprecated`（field option）：如果设置为true，则表示该字段已弃用，新代码不应使用该字段。 在大多数语言中，这没有实际效果。 在Java中，这将成为@Deprecated注释。 将来，其他特定于语言的代码生成器可能会在字段的访问器上生成弃用注释，这会在编译尝试使用该字段的代码时导致发出警告。 如果任何人都没有使用该字段，并且您希望阻止新用户使用该字段，请考虑使用保留语句替换字段声明。
```
int32 old_field = 6 [deprecated=true];
```
###### 15.1 自定义选项
`Protocol Buffer`还允许您定义和使用自己的选项。 这是大多数人不需要的高级功能。 如果您确实认为需要创建自己的选项，请参阅`Proto2`语言指南以获取详细信息。 请注意，创建自定义选项使用的扩展名仅允许用于`proto3`中的自定义选项。

##### 十六、生成您的类
要生成Java，Python，C++，Go，Ruby，Objective-C或C＃代码，您需要使用`.proto`文件中定义的消息类型，您需要在`.proto`上运行`Protocol Buffer`编译器`protoc`。 如果尚未安装编译器，请下载该软件包并按照自述文件中的说明进行操作。 对于Go，您还需要为编译器安装一个特殊的代码生成器插件：您可以在GitHub上的`golang/protobuf`存储库中找到这个和安装说明。

协议编译器的调用如下：
```
protoc --proto_path=IMPORT_PATH --cpp_out=DST_DIR --java_out=DST_DIR --python_out=DST_DIR --go_out=DST_DIR --ruby_out=DST_DIR --objc_out=DST_DIR --csharp_out=DST_DIR path/to/file.proto
```
- `IMPORT_PATH`指定在解析导入指令时在其中查找`.proto`文件的目录。 如果省略，则使用当前目录。 可以通过多次传递`--proto_path`选项来指定多个导入目录; 他们将按顺序搜索。 `-I=IMPORT_PATH`可以用作`--proto_path`的缩写形式。
- 您可以提供一个或多个输出指令,为方便起见，如果`DST_DIR`以`.zip`或`.jar`结尾，编译器会将输出写入具有给定名称的单个ZIP格式存档文件。 `.jar`输出还将根据Java JAR规范的要求提供清单文件。 请注意，如果输出存档已存在，则会被覆盖; 编译器不够智能，无法将文件添加到现有存档中。
- 您必须提供一个或多个`.proto`文件作为输入。 可以一次指定多个`.proto`文件。 虽然文件是相对于当前目录命名的，但每个文件必须驻留在其中一个`IMPORT_PATH`中，以便编译器可以确定其规范名称。
