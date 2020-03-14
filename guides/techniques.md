# 技巧

本页描述了一些常用的`Protocol Buffer`的设计模式。 您还可以将设计和使用问题发送到[`Protocol Buffer`讨论组](https://groups.google.com/forum/#!forum/protobuf)。

## 流式传输多条消息
如果要将多条消息写入单个文件或流，则需要跟踪一条消息的结束位置和下一条消息的开始位置。 `Protocol Buffer`有线格式不是自定界限的，因此`Protocol Buffer`解析器无法确定消息自身的结束位置。 解决此问题的最简单方法是在编写消息本身之前写入每条消息的大小。 当您重新读取消息时，读取大小，然后将字节读入单独的缓冲区，然后从该缓冲区解析。 （如果你想避免将字节复制到一个单独的缓冲区，请查看`CodedInputStream`类（在`C++`和`Java`中），可以告诉它将读取限制为一定的字节数。）

## 大数据集
`Protocol Buffer`不是为处理大型消息而设计的。作为一般经验法则，如果您正在处理大于每兆字节的消息，则可能需要考虑替代策略。

也就是说，`Protocol Buffer`非常适合处理大型数据集中的单个消息。通常，大型数据集实际上只是一小部分的集合，其中每个小部分可能是结构化的数据。尽管`Protocol Buffer`无法同时处理整个集合，但使用`Protocol Buffer`对每个部分进行编码可以极大地简化您的问题：现在您只需要处理一组字节字符串而不是一组结构。

`Protocol Buffer`不包括对大型数据集的任何内置支持，因为不同的情况需要不同的解决方案。有时一个简单的记录列表会做，而有时你可能想要更像数据库的东西。每个解决方案都应该作为一个单独的库开发，以便只有需要它的人才需要支付成本。

## 自描述消息
`Protocol Buffer`不包含其自身类型的描述。 因此，只给出没有定义其类型的相应`.proto`文件的原始消息，很难提取任何有用的数据。

但是，请注意`.proto`文件的内容本身可以使用`Protocol Buffer`表示。 源代码包中的文件`src/google/protobuf/descriptor.proto`定义了所涉及的消息类型。 `protoc`可以使用`--descriptor_set_out`选项输出`FileDescriptorSet`（表示一组`.proto`文件）。 有了这个，您可以定义一个自描述协议消息，如下所示：
```proto
syntax = "proto3";

import "google/protobuf/any.proto";
import "google/protobuf/descriptor.proto";

message SelfDescribingMessage {
  // Set of FileDescriptorProtos which describe the type and its dependencies.
  google.protobuf.FileDescriptorSet descriptor_set = 1;

  // The message and its type, encoded as an Any message.
  google.protobuf.Any message = 2;
}
```
通过使用`DynamicMessage`（可在C ++和Java中使用）这样的类，您可以编写可以操作`SelfDescribingMessages`的工具。

总而言之，这个功能未包含在协议缓冲库中的原因是因为我们从未在Google内部使用过它。

此技术需要使用描述符支持动态消息。 在使用自描述消息之前，请检查您的平台是否支持此功能。
