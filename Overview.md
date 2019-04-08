欢迎使用`protocol buffer`的开发人员文档 - 一种与语言无关，平台无关，可扩展的序列化结构化数据的方法，用于通信协议，数据存储等。

本文档面向希望在其应用程序中使用`protocol buffer`的Java，C ++或Python开发人员。 本概述介绍了`protocol buffer`区，并告诉您需要做什么才能开始 - 然后您可以继续学习本教程或深入研究`protocol buffer`编码。 还为所有三种语言提供API参考文档，以及用于编写`.proto`文件的语言和样式指南。

##### 一、什么是`protocol buffer`？
`protocol buffer`是一种灵活，高效，自动化的机制，用于序列化结构化数据 - 想想XML，但更小，更快，更简单。 您可以定义数据的结构化结构，然后使用特殊生成的源代码轻松地将结构化数据写入和读取各种数据流，并使用各种语言。 您甚至可以更新数据结构，而不会破坏根据“旧”格式编译的已部署程序。

##### 二、`protocol buffer`的工作机制？
您可以通过在`.proto`文件中定义`protocol buffer`消息类型来指定您希望如何构建序列化信息。 每个`protocol buffer`消息都是一个小的逻辑信息记录，包含一系列名称 - 值对。 以下是`.proto`文件的一个非常基本的示例，该文件定义了包含有关人员信息的消息：
```
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

  repeated PhoneNumber phone = 4;
}
```
如您所见，消息格式很简单 - 每种消息类型都有一个或多个唯一编号的字段，每个字段都有一个名称和一个值类型，其中值类型可以是数字（整数或浮点数），布尔值，字符串，原始字节，甚至（如上例所示）其他`protocol buffer`消息类型，允许您分层次地构建数据。您可以指定可选字段，必填字段和重复字段。您可以在`Protocol Buffer Language Guide`中找到有关编写`.proto`文件的更多信息。

一旦定义了消息，就可以在`.proto`文件上运行应用程序语言的`protocol buffer`编译器来生成数据访问类。这些为每个字段提供了简单的访问器（如`name()`和`set_name()`），以及将整个结构序列化/解析为原始字节的方法 - 例如，如果您选择的语言是C++，则运行编译器上面的例子将生成一个名为`Person`的类。然后，您可以在应用程序中使用此类来填充，序列化和检索`Person protocol buffer`消息。然后你可以写一些这样的代码：
```
Person person;
person.set_name("John Doe");
person.set_id(1234);
person.set_email("jdoe@example.com");
fstream output("myfile", ios::out | ios::binary);
person.SerializeToOstream(&output);
```
然后，稍后，您可以在以下位置阅读您的消息：
```
fstream input("myfile", ios::in | ios::binary);
Person person;
person.ParseFromIstream(&input);
cout << "Name: " << person.name() << endl;
cout << "E-mail: " << person.email() << endl;
```
您可以在不破坏向后兼容性的情况下为邮件格式添加新字段; 旧的二进制文件在解析时只是忽略新字段。 因此，如果您的通信协议使用`protocol buffer`作为其数据格式，则可以扩展协议，而无需担心破坏现有代码。

您将在API参考部分找到有关使用生成的`protocol buffer`代码的完整参考，您可以在`protocol buffer`编码中找到有关`protocol buffer`消息如何编码的更多信息。

##### 三、为什么不使用XML？
对于序列化结构化数据，`protocol buffer`比XML具有许多优点。 协议缓冲区：
- 更简单
- 比小3到10倍
- 比你快20到100倍
- 不那么暧昧
- 生成更易于以编程方式使用的数据访问类
例如，假设您想要为具有姓名和电子邮件的人建模。 在XML中，您需要：
```
  <person>
    <name>John Doe</name>
    <email>jdoe@example.com</email>
  </person>
```
而相应的协议缓冲消息（`protocol buffer`文本格式）是：
```
# Textual representation of a protocol buffer.
# This is *not* the binary format used on the wire.
person {
  name: "John Doe"
  email: "jdoe@example.com"
}
```
当此消息被编码为`protocol buffer`二进制格式（上面的文本格式只是用于调试和编辑的方便的人类可读表示）时，它可能是28字节长并且需要大约100-200纳秒来解析。 如果删除空格，XML版本至少为69个字节，并且需要大约5,000-10,000纳秒才能解析。

此外，操作`protocol buffer`要容易得多：
```
  cout << "Name: " << person.name() << endl;
  cout << "E-mail: " << person.email() << endl;
```
而使用XML，您必须执行以下操作：
```
  cout << "Name: "
       << person.getElementsByTagName("name")->item(0)->innerText()
       << endl;
  cout << "E-mail: "
       << person.getElementsByTagName("email")->item(0)->innerText()
       << endl;
```
但是，`protocol buffer`并不总是比XML更好的解决方案 - 例如，`protocol buffer`不是使用标记（例如HTML）对基于文本的文档建模的好方法，因为您无法轻松地将结构与文本交错。 此外，XML是人类可读和人类可编辑的; `protocol buffer`，至少是它们的原生格式，不是。 XML在某种程度上也是自我描述的。 只有拥有消息定义（`.proto`文件）时，`protocol buffer`才有意义。

##### 四、对我来说听起来像是解决方案！ 我该如何开始？
[下载包](https://developers.google.com/protocol-buffers/docs/downloads?hl=zh-CN) - 它包含Java，Python和C ++协议缓冲区编译器的完整源代码，以及I/ O和测试所需的类。 要构建和安装编译器，请按照自述文件中的说明进行操作。

完成所有设置后，请尝试按照所选语言的[教程](https://developers.google.com/protocol-buffers/docs/tutorials?hl=zh-CN)进行操作 - 这将指导您创建一个使用协议缓冲区的简单应用程序。

##### 五、介绍proto3
我们最新的版本3发布了一个新的语言版本-`Protocol Buffer`语言版本3（`aka proto3`），以及我们现有语言版本（`aka proto2`）中的一些新功能。 `Proto3`简化了`Protocol Buffer`，既易于使用，又可以在更广泛的编程语言中使用：我们当前的版本允许您使用Java，C ++，Python，Java Lite，Ruby，JavaScript，Objective-生成`Protocol Buffer`代码。 C和C＃。此外，您可以使用最新的Go protoc插件为Go生成`proto3`代码，该插件可从`golang/protobuf` Github存储库获得。更多语言正在筹备中。

请注意，两种语言版本的API不完全兼容。为避免给现有用户带来不便，我们将继续在新`Protocol Buffer`版本中支持以前的语言版本。

您可以在发行说明中看到与当前默认版本的主要差异，并了解`Proto3`语言指南中的`proto3`语法。 `proto3`的完整文档即将推出！

（如果名称`proto2`和`proto3`看起来有点令人困惑，那是因为当我们最初开源的`Protocol Buffer`时，它实际上是Google的第二版语言 - 也称为`proto2`。这也是我们的开源版本号从v2.0.0开始的原因。)

##### 六、一些历史
`Protocol Buffer`最初是在Google开发的，用于处理索引服务器请求/响应协议。 在`Protocol Buffer`之前，有一种请求和响应的格式，它使用请求和响应的手动编组/解组，并支持许多版本的协议。 这导致了一些非常丑陋的代码，例如：
```
 if (version == 3) {
   ...
 } else if (version > 4) {
   if (version == 5) {
     ...
   }
   ...
 }
```
明确格式化的协议也使新协议版本的推出变得复杂，因为开发人员必须确保请求的发起者和处理请求的实际服务器之间的所有服务器都能理解新协议，然后才能切换开关以开始使用新协议。

`Protocol Buffer`旨在解决许多这些问题：
- 可以轻松引入新字段，而不需要检查数据的中间服务器可以简单地解析它并传递数据而无需了解所有字段。
- 格式更具自我描述性，可以用各种语言（C++，Java等）处理。

但是，用户仍然需要手写自己的解析代码。

随着系统的发展，它获得了许多其他功能和用途：

- 自动生成的序列化和反序列化代码避免了手动解析的需要。
- 除了用于短期RPC（远程过程调用）请求之外，人们还开始使用`Protocol Buffer`作为一种方便的自描述格式，用于持久存储数据（例如，在Bigtable中）。
- 服务器RPC接口开始被声明为协议文件的一部分，协议编译器生成存根类，用户可以通过服务器接口的实际实现来覆盖它们。

`Protocol Buffer`现在是Google的数据通用语言 - 在撰写本文时，Google代码树中有12,183个`.proto`文件中定义了48,162种不同的消息类型。 它们既可用于RPC系统，也可用于各种存储系统中的数据持久存储。
