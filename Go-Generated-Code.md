此页面准确描述了`protocol buffer`编译器为任何给定协议定义生成的Go代码。 `proto2`和`proto3`生成代码之间的任何差异都会突出显示 - 请注意，这些差异在本文档中描述的生成代码中，而不是基本API，两个版本中的相同。 在阅读本文档之前，您应该阅读`proto2`语言指南和/或`proto3`语言指南。

##### 一、编译器调用
`protocol buffer`编译器需要一个[插件](https://github.com/golang/protobuf)来生成Go代码。 用它安装
```
go get github.com/golang/protobuf/protoc-gen-go
```
提供了`protoc-gen-go`二进制文件，`protoc`在使用`--go_out`命令行标志调用时使用。 `--go_out`标志告诉编译器在哪里写Go源文件。 编译器为每个`.proto`文件输入创建单个源文件。

输出文件的名称是通过获取`.proto`文件的名称并进行两处更改来计算的：

- 扩展名（`.proto`）替换为`.pb.go`。 例如，名为`player_record.proto`的文件会生成一个名为`player_record.pb.go`的输出文件。
- `proto`路径（使用`--proto_path`或`-I`命令行标志指定）将替换为输出路径（使用`--go_out`标志指定）。

当你像这样运行`proto`编译器时：
```
protoc --proto_path=src --go_out=build/gen src/foo.proto src/bar/baz.proto
```
编译器将读取文件`src/foo.proto`和`src/bar/baz.proto`。 它产生两个输出文件：`build/gen/foo.pb.go`和`build/gen/bar/baz.pb.go`。

如有必要，编译器会自动创建目录`build/gen/bar`，但不会创建`build`或`build/gen`; 他们必须已经存在。

##### 二。包
如果`.proto`文件包含包声明，则生成的代码将使用`proto`的包作为其Go包名称进行转换。 首先将字符转换为`_` 例如，`example.high_score`的`proto`包名称导致Go包名称为`example_high_score`。

您可以使用`.proto`文件中的`go_package`选项覆盖特定`.proto`的默认生成包。 例如，包含的`.proto`文件
```
package example.high_score;
option go_package = "hs";
```
使用Go包名称`hs`生成一个文件。
否则，如果`.proto`文件不包含包声明，则生成的代码使用文件名（减去扩展名）作为Go包名称进行转换。 首先将字符转换为`_` 例如，没有包声明的名为`high.score.proto`的`proto`包将生成包含`high_score`的文件`high.score.pb.go`。

##### 三、消息
给出一个简单的消息声明：
```
message Foo {}
```
`protocol buffer`编译器生成一个名为`Foo`的结构。 A `*Foo`实现了`Message`接口。 有关详细信息，请参阅内联注释。
```
type Foo struct {
}

// Reset sets the proto's state to default values.
func (m *Foo) Reset()         { *m = Foo{} }

// String returns a string representation of the proto.
func (m *Foo) String() string { return proto.CompactTextString(m) }

// ProtoMessage acts as a tag to make sure no one accidentally implements the
// proto.Message interface.
func (*Foo) ProtoMessage()    {}
```
请注意，所有这些成员始终存在; `optimize_for`选项不会影响Go代码生成器的输出。

###### 3.1 嵌套类型
可以在另一条消息中声明消息。 例如：
```
message Foo {
  message Bar {
  }
}
```
在这种情况下，编译器生成两个结构：`Foo`和`Foo_Bar`。

###### 3.2 知名类型
`Protobufs`带有一组预定义的消息，称为众所周知的类型（WKT）。 这些类型可以用于与其他服务的互操作性，或者仅仅因为它们简洁地表示常见的有用模式。 例如，`Struct`消息表示任意C样式结构的格式。

WKT的预生成Go代码作为Go protobuf库的一部分进行分发，如果使用WKT，则生成的消息的Go代码会引用此代码。 例如，给出如下消息：
```
import "google/protobuf/struct.proto"
import "google/protobuf/timestamp.proto"

message NamedStruct {
  string name = 1;
  google.protobuf.Struct definition = 2;
  google.protobuf.Timestamp last_modified = 3;
}
```
生成的Go代码如下所示：
```
import google_protobuf "github.com/golang/protobuf/ptypes/struct"
import google_protobuf1 "github.com/golang/protobuf/ptypes/timestamp"

...

type NamedStruct struct {
   Name         string
   Definition   *google_protobuf.Struct
   LastModified *google_protobuf1.Timestamp
}
```
一般来说，您不需要将这些类型直接导入代码中。 但是，如果需要直接引用其中一种类型，只需导入`github.com/golang/protobuf/ptypes/[TYPE]`包，并正常使用该类型。

##### 四、字段
`protocol buffer`编译器为消息中定义的每个字段生成结构字段。 该字段的确切性质取决于其类型以及它是单个，重复，映射还是单个字段。

请注意，生成的Go字段名称始终使用驼峰式命名，即使`.proto`文件中的字段名称使用带有下划线的小写字母（应该如此）。 案例转换的工作原理如下：

- 第一个字母为出口资本化。如果第一个字符是下划线，则将其删除并添加大写字母X.
- 如果内部下划线后跟小写字母，则删除下划线，并将以下字母大写。

因此，原型字段`foo_bar_baz`在Go中变为`FooBarBaz`，而`_my_field_name_2`变为`XMyFieldName_2`。

###### 4.1 单数标量字段（proto2）
对于以下任一字段定义：
```
optional int32 foo = 1;
required int32 foo = 1;
```
编译器生成一个结构，其中包含一个名为`Foo`的`* int32`字段和一个访问器方法`GetFoo()`，它返回`Foo`中的`int32`值或默认值（如果该字段未设置）。 如果未显式设置默认值，则使用该类型的零值（0表示数字，空字符串表示字符串）。

对于其他标量字段类型（包括`bool`，`bytes`和`string`），`* int32`将根据标量值类型表替换为相应的Go类型。

###### 4.2 单数标量字段（proto3）
对于此字段定义：
```
int32 foo = 1;
```
编译器将生成一个带有名为`Foo`的`int32`字段和一个访问器方法`GetFoo()`的结构，该方法返回`Foo`中的`int32`值或该字段的零值（如果字段未设置）（数字为0，字符串为空字符串）。

对于其他标量字段类型（包括`bool`，`bytes`和`string`），根据标量值类型表将`int32`替换为相应的Go类型。 `proto`中的未设置值将表示为该类型的零值（0表示数字，空字符串表示字符串）。

###### 4.3 单数消息字段
指定消息类型：
```
message Bar {}
```
此消息有`Bar`字段:
```
// proto2
message Baz {
  optional Bar foo = 1;
  // The generated code is the same result if required instead of optional.
}

// proto3
message Baz {
  Bar foo = 1;
}
```
编译器将产生Go的结构体
```
type Baz struct {
        Foo *Bar
}
```
消息字段可以设置为`nil`，这意味着该字段未设置，有效清除该字段。 这不等同于将值设置为消息`struct`的"空"实例。

编译器还生成一个`func（m *Baz）GetFoo() *Bar)辅助函数。 这使得可以在没有中间零检查的情况下链接获取调用。

###### 4.4 `repeated`字段
每个重复的字段在Go中的结构中生成一个`T`字段，其中`T`是字段的元素类型。 对于带有重复字段的此消息：
```
message Baz {
  repeated Bar foo = 1;
}
```
编译器将产生Go的结构体：
```
type Baz struct {
        Foo  []*Bar
}
```
同样，对于字段定义，重复字节`foo = 1`; 编译器将生成一个带有名为`Foo`的`[] []`字节字段的Go结构。 对于重复的枚举重复`MyEnum bar = 2`;，编译器生成一个带有名为`Bar`的`[] MyEnum`字段的结构。

###### 4.5 `Map`字段
每个`map`字段在类型`map[TKey]TValue`的结构中生成一个字段，其中`TKey`是字段的键类型，`TValue`是字段的值类型。 对于带有`map`字段的此消息：
```
message Bar {}

message Baz {
  map<string, Bar> foo = 1;
}
```
编译器将生成Go结构体：
```
type Baz struct {
        Foo map[string]*Bar
}
```
###### 4.6 `Oneof`字段
对于`oneof`字段，`protobuf`编译器生成具有接口类型`isMessageName_MyField`的单个字段。 它还为`oneof`中的每个奇异字段生成一个结构。 这些都实现了这个`isMessageName_MyField`接口。

对于带有`oneof`字段的此消息：
```
package account;
message Profile {
  oneof avatar {
    string image_url = 1;
    bytes image_data = 2;
  }
}
```
编译器生成结构体：
```
type Profile struct {
        // Types that are valid to be assigned to Avatar:
        //      *Profile_ImageUrl
        //      *Profile_ImageData
        Avatar isProfile_Avatar `protobuf_oneof:"avatar"`
}

type Profile_ImageUrl struct {
        ImageUrl string
}
type Profile_ImageData struct {
        ImageData []byte
}
```
`*Profile_ImageUrl`和`*Profile_ImageData`都通过提供空的`isProfile_Avatar()`方法来实现`isProfile_Avatar`。

以下示例显示如何设置字段：
```
p1 := &account.Profile{
  Avatar: &account.Profile_ImageUrl{"http://example.com/image.png"},
}

// imageData is []byte
imageData := getImageData()
p2 := &account.Profile{
  Avatar: &account.Profile_ImageData{imageData},
}
```
要访问该字段，您可以使用值上的类型开关来处理不同的消息类型。
```
switch x := m.Avatar.(type) {
case *account.Profile_ImageUrl:
        // Load profile image based on URL
        // using x.ImageUrl
case *account.Profile_ImageData:
        // Load profile image based on bytes
        // using x.ImageData
case nil:
        // The field is not set.
default:
        return fmt.Errorf("Profile.Avatar has unexpected type %T", x)
}
```
编译器还生成`get`方法`func (m *Profile) GetImageUrl()`字符串和`func (m *Profile) GetImageData) []`字节。 每个get函数返回该字段的值，如果未设置则返回零值。

##### 五、枚举
给出如下枚举：
```
message SearchRequest {
  enum Corpus {
    UNIVERSAL = 0;
    WEB = 1;
    IMAGES = 2;
    LOCAL = 3;
    NEWS = 4;
    PRODUCTS = 5;
    VIDEO = 6;
  }
  Corpus corpus = 1;
  ...
}
```
`protocol buffer`编译器生成一种类型和一系列具有该类型的常量。

对于消息中的枚举（如上面的那个），类型名称以消息名称开头：
```
type SearchRequest_Corpus int32
```
对于包级别的枚举：
```
enum Foo {
  DEFAULT_BAR = 0;
  BAR_BELLS = 1;
  BAR_B_CUE = 2;
}
```
他的go类型名称未从`proto` 枚举名称修改：
```
type Foo int32
```
此类型具有`String()`方法，该方法返回给定值的名称。

`Enum()`方法使用给定值初始化新分配的内存并返回相应的指针：
```
func (Foo) Enum() *Foo
```
`protocol buffer`编译器为枚举中的每个值生成一个常量。 对于消息中的枚举，常量以封闭消息的名称开头：
```
const (
        SearchRequest_UNIVERSAL SearchRequest_Corpus = 0
        SearchRequest_WEB       SearchRequest_Corpus = 1
        SearchRequest_IMAGES    SearchRequest_Corpus = 2
        SearchRequest_LOCAL     SearchRequest_Corpus = 3
        SearchRequest_NEWS      SearchRequest_Corpus = 4
        SearchRequest_PRODUCTS  SearchRequest_Corpus = 5
        SearchRequest_VIDEO     SearchRequest_Corpus = 6
)
```
对于包级别的枚举，常量以枚举名称开头：
```
const (
        Foo_DEFAULT_BAR Foo = 0
        Foo_BAR_BELLS   Foo = 1
        Foo_BAR_B_CUE   Foo = 2
)
```
`protobuf`编译器还生成从整数值到字符串名称的映射以及从名称到值的映射：
```
var Foo_name = map[int32]string{
        0: "DEFAULT_BAR",
        1: "BAR_BELLS",
        2: "BAR_B_CUE",
}
var Foo_value = map[string]int32{
        "DEFAULT_BAR": 0,
        "BAR_BELLS":   1,
        "BAR_B_CUE":   2,
}
```
请注意，`.proto`语言允许多个枚举符号具有相同的数值。 具有相同数值的符号是同义词。 这些在Go中以完全相同的方式表示，多个名称对应于相同的数值。 反向映射包含数字值的单个条目，该条目首先出现在`.proto`文件中。

##### 六、扩展
扩展仅存在于`proto2`中。 有关`proto2`扩展的Go生成代码API的文档，请参阅[proto包doc](https://godoc.org/github.com/golang/protobuf/proto)。
##### 七、服务
默认情况下，Go代码生成器不会为服务生成输出。 如果您启用gRPC插件（请参阅[gRPC Go快速入门指南](https://github.com/grpc/grpc-go/tree/master/examples)），则会生成代码以支持gRPC。

