# `protocol buffer`教程：`Golang`

本教程使用`proto3`版本的`protocol buffers`语言，提供了一个基本的`Go`程序员使用`protocol buffers`的介绍。 通过创建一个简单的示例应用程序，它会告诉您如何操作:
- 在`.proto`文件中定义`message`格式。
- 使用`protocol buffers`编译器
- 使用`Go`的`protocol buffers API`读写消息

这不是在`Go`中使用`protocol buffers`的综合指南。 有关更详细的参考信息，请参阅[`protocol buffer`语言指南](https://developers.google.com/protocol-buffers/docs/proto3?hl=zh-CN)，[`Go API`参考](https://godoc.org/github.com/golang/protobuf/proto)，[`Go`生成代码指南](https://developers.google.com/protocol-buffers/docs/reference/go-generated?hl=zh-CN)和[编码参考](https://developers.google.com/protocol-buffers/docs/encoding?hl=zh-CN)。

## 为什么使用protocol buffers？
我们将要使用的示例是一个非常简单的"地址簿"应用程序，可以在文件中读取和写入人员的联系人详细信息。 地址簿中的每个人都有姓名，`ID`，电子邮件地址和联系电话号码。

你如何序列化和检索这样的结构化数据？ 有几种方法可以解决这个问题：
- 使用`gobs`序列化`Go`数据结构。 这是`Go`特定环境中的一个很好的解决方案，但如果您需要与为其他平台编写的应用程序共享数据，它将无法正常工作。
- 您可以发明一种特殊的方法将数据项编码为单个字符串 - 例如将4个整数编码为“12：3：-23：67”。 这是一种简单而灵活的方法，虽然它确实需要编写一次性编码和解析代码，并且解析会产生很小的运行时成本。 这最适合编码非常简单的数据。
- 将数据序列化为`XML`。 这种方法非常有吸引力，因为`XML`（有点）是人类可读的，并且有许多语言的绑定库。 如果您想与其他应用程序/项目共享数据，这可能是一个不错的选择。 然而，`XML`是众所周知的空间密集型，并且编码/解码它会对应用程序造成巨大的性能损失。 此外，导航`XML DOM`树比通常在类中导航简单字段要复杂得多。

`protocol buffers`是灵活，高效，自动化的解决方案，可以解决这个问题。 使用`protocol buffers`，您可以编写要存储的数据结构的`.proto`描述。 由此，`protocol buffers`编译器创建一个类，该类使用有效的二进制格式实现`protocol buffers`数据的自动编码和解析。 生成的类为构成`protocol buffers`的字段提供`getter`和`setter`，并负责将`protocol buffers`作为一个单元读取和写入的细节。 重要的是，`protocol buffers`格式支持随着时间的推移扩展格式的想法，使得代码仍然可以读取用旧格式编码的数据。

## 哪里找到示例代码？
我们的示例是一组用于管理地址簿数据文件的命令行应用程序，使用`protocol buffers`区进行编码。 命令`add_person_go`向数据文件添加新条目。 命令`list_people_go`解析数据文件并将数据打印到控制台。

您可以在`GitHub`存储库的[`examples`目录](https://github.com/protocolbuffers/protobuf/tree/master/examples)中找到完整的示例。

## 定义协议格式
要创建地址簿应用程序，您需要从`.proto`文件开始。 `.proto`文件中的定义很简单：为要序列化的每个数据结构添加消息，然后为消息中的每个字段指定名称和类型。 在我们的示例中，定义消息的`.proto`文件是`addressbook.proto`。

`.proto`文件以包声明开头，这有助于防止不同项目之间的命名冲突。
```proto
syntax = "proto3";
package tutorial;

import "google/protobuf/timestamp.proto";
```
在`Go`中，`package`名称用作`Go`包，除非您指定了`go_package`。 即使你确实提供了`go_package`，你仍然应该定义一个普通的包，以避免在`Protocol Buffers`名称空间和非`Go`语言中发生名称冲突。

接下来，您有`message`定义。 `message`只是包含一组类型字段的聚合。 许多标准的简单数据类型都可用作字段类型，包括`bool`，`int32`，`float`，`double`和`string`。 您还可以使用其他消息类型作为字段类型，为邮件添加更多结构。
```proto
message Person {
  string name = 1;
  int32 id = 2;  // Unique ID number for this person.
  string email = 3;

  enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
  }

  message PhoneNumber {
    string number = 1;
    PhoneType type = 2;
  }

  repeated PhoneNumber phones = 4;

  google.protobuf.Timestamp last_updated = 5;
}

// Our address book file is just one of these.
message AddressBook {
  repeated Person people = 1;
}
```
在上面的示例中，`Person`消息包含`PhoneNumber`消息，而`AddressBook`消息包含`Person`消息。 您甚至可以定义嵌套在其他消息中的消息类型 - 如您所见，`PhoneNumber`类型在`Person`中定义。 如果您希望其中一个字段具有预定义的值列表之一，您还可以定义枚举类型 - 此处您要指定电话号码可以是`MOBILE`，`HOME`或`WORK`之一。

每个元素上的"= 1"，"= 2"标记标识该字段在二进制编码中使用的唯一"标记"。 标签号1-15需要少于一个字节来编码而不是更高的数字，因此作为优化，您可以决定将这些标签用于常用或重复的元素，将标签16和更高版本留给不太常用的可选元素。 重复字段中的每个元素都需要重新编码标记号，因此重复字段特别适合此优化。

如果未设置字段值，则使用[默认值](https://developers.google.com/protocol-buffers/docs/proto3?hl=zh-CN#default)：数字类型为零，字符串为空字符串，bools为false。 对于嵌入式消息，默认值始终是消息的"默认实例"或"原型"，其中没有设置其字段。 调用访问器以获取尚未显式设置的字段的值始终返回该字段的默认值。

如果一个字段是`repeated`，该字段可以重复任意次数（包括零）。 重复值的顺序将保留在`Protocol Buffers`中。 将重复字段视为动态大小的数组。

您将在[Protocol Buffer Language Guide](https://developers.google.com/protocol-buffers/docs/proto3?hl=zh-CN)中找到编写`.proto`文件的完整指南 - 包括所有可能的字段类型。 不要去寻找类继承类似的工具，但协议缓冲区不会这样做。

## 编译Protocol Buffers
既然你有一个`.proto`，你需要做的下一件事是生成你需要读取和写入`AddressBook`（以及`Person`和`PhoneNumber`）消息所需的类。 为此，您需要在`.proto`上运行`Protocol Buffers`编译器`protoc`：
1. 如果尚未安装编译器，请[下载该软件包](https://developers.google.com/protocol-buffers/docs/downloads?hl=zh-CN)并按照`README`进行操作。
2. 运行以下命令安装`Go Protocol Buffers`插件：
```bash
go get -u github.com/golang/protobuf/protoc-gen-go
```
编译器插件`protoc-gen-go`将安装在`$GOBIN`中，默认为`$GOPATH/bin`。 必须在协议编译器`protoc`的`$PATH`中才能找到它。
3. 现在运行编译器，指定源目录（应用程序的源代码所在的位置 - 如果不提供值，则使用当前目录），目标目录（您希望生成的代码在哪里;通常与`$SRC_DIR`相同），以及`.proto`的路径。 在这种情况下，你...：
```bash
protoc -I=$SRC_DIR --go_out=$DST_DIR $SRC_DIR/addressbook.proto
```
因为您需要`Go`类，所以使用`--go_out`选项 - 为其他支持的语言提供了类似的选项。

这会在指定的目标目录中生成`addressbook.pb.go`。

## `Protocol Buffer API`
生成`addressbook.pb.go`为您提供以下有用类型：
- 具有`People`字段的`AddressBook`结构。
- 具有`Name`，`Id`，`Email`和`Phones`字段的`Person`结构。
- `Person_PhoneNumber`结构，包含`Number`和`Type`字段。
- `Person_PhoneType`类型和为`Person.PhoneType`枚举中的每个值定义的值。
您可以阅读更多有关[生成代码](https://developers.google.com/protocol-buffers/docs/reference/go-generated?hl=zh-CN)指南中生成的内容的详细信息，但在大多数情况下，您可以将这些视为完全普通的Go类型。

这是[`list_people`](https://github.com/protocolbuffers/protobuf/blob/master/examples/list_people_test.go)命令关于如何创建Person实例的单元测试的示例：
```Golang
p := pb.Person{
        Id:    1234,
        Name:  "John Doe",
        Email: "jdoe@example.com",
        Phones: []*pb.Person_PhoneNumber{
                {Number: "555-4321", Type: pb.Person_HOME},
        },
}
```
## 编写`message`
使用`Protocol Buffer`的全部目的是序列化您的数据，以便可以在其他地方解析它。 在`Go`中，您使用`proto`库的[`Marshal`](https://godoc.org/github.com/golang/protobuf/proto#Marshal)函数来序列化协议缓冲区数据。 `Protocol Buffer`消息的`struct`的指针实现了`proto.Message`接口。 调用`proto.Marshal`会返回以其有线格式编码的`Protocol Buffer`。 例如，我们在`add_person`命令中使用此函数：
```Golang
book := &pb.AddressBook{}
// ...

// Write the new address book back to disk.
out, err := proto.Marshal(book)
if err != nil {
        log.Fatalln("Failed to encode address book:", err)
}
if err := ioutil.WriteFile(fname, out, 0644); err != nil {
        log.Fatalln("Failed to write address book:", err)
}
```
## 读取message
要解析编码消息，请使用`proto`库的[`Unmarshal`](https://godoc.org/github.com/golang/protobuf/proto#Unmarshal)函数。 调用它将`buf`中的数据解析为Protocol Buffer，并将结果放在`pb`中。 因此，要在`list_people`命令中解析文件，我们使用：
```Golang
// Read the existing address book.
in, err := ioutil.ReadFile(fname)
if err != nil {
        log.Fatalln("Error reading file:", err)
}
book := &pb.AddressBook{}
if err := proto.Unmarshal(in, book); err != nil {
        log.Fatalln("Failed to parse address book:", err)
}
```

## 扩展`Protocol Buffer`
在释放使用`Protocol Buffer`区的代码之后迟早，您无疑会想要"改进"`Protocol Buffer`的定义。 如果你希望你的新`Protocol Buffer`向后兼容，并且你的旧缓冲区是向前兼容的 - 而且你几乎肯定想要这个 - 那么你需要遵循一些规则。 在新版本的`Protocol Buffer`中：
- 您不得更改任何现有字段的标记号。
- 你可以删除字段。
- 您可以添加新字段，但必须使用新的标记号（即从未在此`Protocol Buffer`中使用的标记号，甚至不包括已删除的字段）。
（这些规则有一些[例外](https://developers.google.com/protocol-buffers/docs/proto3?hl=zh-CN#updating)，但它们很少使用。）

如果您遵循这些规则，旧代码将很乐意阅读新消息并简单地忽略任何新字段。 对于旧代码，已删除的单个字段将只具有其默认值，删除的重复字段将为空。 新代码也将透明地读取旧消息。

但是，请记住旧消息中不会出现新字段，因此您需要使用默认值执行合理的操作。 使用特定于类型的默认值：对于字符串，默认值为空字符串。 对于布尔值，默认值为`false`。 对于数字类型，默认值为零。
