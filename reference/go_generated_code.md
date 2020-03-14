# 生成`Go`代码

此页面准确描述了`protocol buffer`编译器为任何给定协议定义生成的`Go`代码。 `proto2`和`proto3`生成代码之间的任何差异都会突出显示 - 请注意，这些差异在本文档中描述的生成代码中，而不是基本`API`，两个版本中的相同。 在阅读本文档之前，您应该阅读`proto2`语言指南和/或`proto3`语言指南。

## 编译器调用
`protocol buffer`编译器需要一个[插件](https://github.com/golang/protobuf)来生成`Go`代码。 用它安装
```bash
go get github.com/golang/protobuf/protoc-gen-go
```

这将在`$GOBIN`安装`protoc-gen-go`二进制文件。设置`$GOBIN`环境变量以更改安装位置。它必须是在你`$PATH`的`protocol buffer`编译器找到它。使用该`--go_out`标志调用时，`protocol buffer`编译器将生成`Go`输出 。该`--go_out`标志的参数是您希望编译器在其中写入`Go`输出的目录。编译器为每个`.proto`文件输入创建一个源文件。通过将`.proto`扩展名替换为来创建输出文件的名称`.pb.go`。`.proto`文件应包含一个`go_packag`e选项，用于指定包含所生成代码的`Go`软件包的完整导入路径。
```proto
option go_package ="example.com/foo/bar";
```
输出文件所在的输出目录的子目录取决于`go_package`选项和编译器标志：
- 默认情况下，输出文件放置在以`Go`软件包的导入路径命名的目录中。例如，`protos/foo.proto` 具有上述`go_package`选项的文件将生成名为的文件`example.com/foo/bar/foo.pb.go`。
- 如果给`--go_opt=paths=source_relative`标记`protoc`，则将输出文件放置在与输入文件相同的相对目录中。例如，该文件`protos/foo.proto` 生成名为的文件`protos/foo.pb.go`。

当你像这样运行`proto`编译器时：
```
protoc --proto_path=src --go_out=build/gen --go_opt=paths=source_relative src/foo.proto src/bar/baz.proto
```
编译器将读取文件`src/foo.proto`和`src/bar/baz.proto`。 它产生两个输出文件：`build/gen/foo.pb.go`和`build/gen/bar/baz.pb.go`。

如有必要，编译器会自动创建目录`build/gen/bar`，但不会创建`build`或`build/gen`; 他们必须已经存在。

## 包
源`.proto`文件应包含一个`go_package`选项，用于指定文件的完整`Go`导入路径。如果没有`go_package` 选择，编译器将尝试猜测一个。编译器的未来版本将使该`go_package`选项成为必需。生成代码的`Go`软件包名称将是该`go_package`选项的最后一个路径部分 。

## 消息
给出一个简单的消息声明：
```proto
message Foo {}
```

`protovol buffer`编译器生成一个名为的结构`Foo`。一个`*Foo`实现`proto.Message`接口。

该`proto`软件包 提供了对消息进行操作的功能，包括与二进制格式之间的转换。

该`proto.Message`接口定义了一个`ProtoReflec`t方法。此方法返回`protoreflect.Message` ，提供消息的基于反射的视图。

该`optimize_for`选项不会影响`Go`代码生成器的输出。

### 嵌套类型
可以在另一条消息中声明消息。 例如：
```
message Foo {
  message Bar {
  }
}
```
在这种情况下，编译器生成两个结构：`Foo`和`Foo_Bar`。

## 字段
`protocol buffer`编译器为消息中定义的每个字段生成结构字段。 该字段的确切性质取决于其类型以及它是单个，重复，映射还是单个字段。

请注意，生成的`Go`字段名称始终使用驼峰式命名，即使`.proto`文件中的字段名称使用带有下划线的小写字母（应该如此）。 案例转换的工作原理如下：

- 第一个字母为出口资本化。如果第一个字符是下划线，则将其删除并添加大写字母`X`.
- 如果内部下划线后跟小写字母，则删除下划线，并将以下字母大写。

因此，原型字段`foo_bar_baz`在Go中变为`FooBarBaz`，而`_my_field_name_2`变为`XMyFieldName_2`。

### 单数标量字段（proto2）
对于以下任一字段定义：
```proto2
optional int32 foo = 1;
required int32 foo = 1;
```
编译器生成一个结构，其中包含一个名为`Foo`的`* int32`字段和一个访问器方法`GetFoo()`，它返回`Foo`中的`int32`值或默认值（如果该字段未设置）。 如果未显式设置默认值，则使用该类型的零值（0表示数字，空字符串表示字符串）。

对于其他标量字段类型（包括`bool`，`bytes`和`string`），`* int32`将根据标量值类型表替换为相应的Go类型。

### 单数标量字段（proto3）
对于此字段定义：
```proto3
int32 foo = 1;
```
编译器将生成一个带有名为`Foo`的`int32`字段和一个访问器方法`GetFoo()`的结构，该方法返回`Foo`中的`int32`值或该字段的零值（如果字段未设置）（数字为0，字符串为空字符串）。

对于其他标量字段类型（包括`bool`，`bytes`和`string`），根据标量值类型表将`int32`替换为相应的Go类型。 `proto`中的未设置值将表示为该类型的零值（0表示数字，空字符串表示字符串）。

### 单数消息字段
指定消息类型：
```proto
message Bar {}
```
此消息有`Bar`字段:
```proto
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
编译器将产生`Go`的结构体
```Golang
type Baz struct {
        Foo *Bar
}
```
消息字段可以设置为`nil`，这意味着该字段未设置，有效清除该字段。 这不等同于将值设置为消息`struct`的"空"实例。

编译器还生成一个`func（m *Baz）GetFoo() *Bar)`辅助函数。 这使得可以在没有中间零检查的情况下链接获取调用。

### `repeated`字段
每个重复的字段在`Go`中的结构中生成一个`T`字段，其中`T`是字段的元素类型。 对于带有重复字段的此消息：
```proto
message Baz {
  repeated Bar foo = 1;
}
```
编译器将产生`Go`的结构体：
```Golang
type Baz struct {
        Foo  []*Bar
}
```
同样，对于字段定义，重复字节`foo = 1`; 编译器将生成一个带有名为`Foo`的`[] []`字节字段的Go结构。 对于重复的枚举重复`MyEnum bar = 2`;，编译器生成一个带有名为`Bar`的`[] MyEnum`字段的结构。

下面的例子展示如何设置该字段
```Golang
baz := &Baz{
  Foo: []*Bar{
    {}, // First element.
    {}, // Second element.
  },
}
```

要访问该字段，可以这样做
```Golang
foo := baz.GetFoo() // foo type is []*Bar.
b1 := foo[0] // b1 type is *Bar, the first element in foo.
```

### `Map`字段
每个`map`字段在类型`map[TKey]TValue`的结构中生成一个字段，其中`TKey`是字段的键类型，`TValue`是字段的值类型。 对于带有`map`字段的此消息：
```proto
message Bar {}

message Baz {
  map<string, Bar> foo = 1;
}
```
编译器将生成`Go`结构体：
```Golang
type Baz struct {
        Foo map[string]*Bar
}
```
### `Oneof`字段
对于`oneof`字段，`protobuf`编译器生成具有接口类型`isMessageName_MyField`的单个字段。 它还为`oneof`中的每个奇异字段生成一个结构。 这些都实现了这个`isMessageName_MyField`接口。

对于带有`oneof`字段的此消息：
```proto
package account;
message Profile {
  oneof avatar {
    string image_url = 1;
    bytes image_data = 2;
  }
}
```
编译器生成结构体：
```Golang
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
```Golang
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
```Golang
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
编译器还生成`get`方法`func (m *Profile) GetImageUrl()`字符串和`func (m *Profile) GetImageData) []`字节。 每个`get`函数返回该字段的值，如果未设置则返回零值。

## 枚举
给出如下枚举：
```proto
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
```Golang
type SearchRequest_Corpus int32
```
对于包级别的枚举：
```proto
enum Foo {
  DEFAULT_BAR = 0;
  BAR_BELLS = 1;
  BAR_B_CUE = 2;
}
```
他的`go`类型名称未从`proto` 枚举名称修改：
```Golang
type Foo int32
```
此类型具有`String()`方法，该方法返回给定值的名称。

`Enum()`方法使用给定值初始化新分配的内存并返回相应的指针：
```Golang
func (Foo) Enum() *Foo
```
`protocol buffer`编译器为枚举中的每个值生成一个常量。 对于消息中的枚举，常量以封闭消息的名称开头：
```Golang
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
```Golang
const (
        Foo_DEFAULT_BAR Foo = 0
        Foo_BAR_BELLS   Foo = 1
        Foo_BAR_B_CUE   Foo = 2
)
```
`protobuf`编译器还生成从整数值到字符串名称的映射以及从名称到值的映射：
```Golang
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
请注意，`.proto`语言允许多个枚举符号具有相同的数值。 具有相同数值的符号是同义词。 这些在`Go`中以完全相同的方式表示，多个名称对应于相同的数值。 反向映射包含数字值的单个条目，该条目首先出现在`.proto`文件中。

## 扩展
扩展仅存在于`proto2`中。 有关`proto2`扩展的Go生成代码API的文档，请参阅[proto包doc](https://godoc.org/github.com/golang/protobuf/proto)。

## 服务
默认情况下，`Go`代码生成器不会为服务生成输出。 如果您启用`gRPC`插件（请参阅[gRPC Go快速入门指南](https://github.com/grpc/grpc-go/tree/master/examples)），则会生成代码以支持`gRPC`。

