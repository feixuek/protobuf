# 样式指南

本文档提供了`.proto`文件样式指南。通过遵循这些约定，可以使`protocol buffer`消息定义及其相应的类保持一致并易于阅读。

请注意，`protocol buffer`样式已随着时间而发展，因此您可能会看到以不同的约定或样式编写的`.proto`文件。 修改这些文件时，请尊重现有样式。一致性是关键。但是，在创建新的`.proto`文件时，最好采用当前的最佳样式。

## 标准文件格式
- 保持行长为80个字符。
- 缩进2个空格。

## 文件结构
文件应命名`lower_snake_case.proto`

所有文件应按以下方式排版：

1. 许可标头（如果适用）
2. 文件总览
3. 句法
4. 包
5. 导入（分类）
6. 文件选项
7. 其他一切

## 包
软件包名称应为小写，并且应与目录层次结构相对应。例如，如果文件在其中`my/package/`，则软件包名称应为`my.package`。

## 消息和字段名称
将`CamelCase`（带有初始大写）用于消息名称 - 例如，`SongServerRequest`。 将`underscore_separated_names`用于字段名称 - 例如，`song_name`。
```proto
message SongServerRequest {
  required string song_name = 1;
}
```
对字段名称使用此命名约定可为您提供以下访问器：
```proto
C++:
  const string& song_name() { ... }
  void set_song_name(const string& x) { ... }

Java:
  public String getSongName() { ... }
  public Builder setSongName(String v) { ... }
```

如果您的字段名称包含数字，则该数字应出现在字母之后而不是下划线之后。例如，使用`song_name1`代替`song_name_1`

## 重复的字段
对重复的字段使用复数名称。
```proto
  repeated string keys = 1;
  ...
  repeated MyMessage accounts = 17;
```
## 枚举
对于枚举类型名称使用`CamelCase`（带有初始大写），对值名称使用`CAPITALS_WITH_UNDERSCORES`：
```proto
enum Foo {
  FIRST_VALUE = 0;
  SECOND_VALUE = 1;
}
```
每个枚举值应以分号结束，而不是逗号。

## 服务
如果`.proto`定义了`RPC`服务，则应该对服务名称和任何`RPC`方法名称使用`CamelCase`（带有初始大写）：
```
service FooService {
  rpc GetSomething(FooRequest) returns (FooResponse);
}
```

## 避免的事情
- 必填字段（仅适用于`proto2`）
- 组（仅适用于`proto2`）
