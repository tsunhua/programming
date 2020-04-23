# Go 中使用 ProtoBuf

## package & import

1. package 名应该与所在文件夹同名，且只使用小写字母，不用大写字母也不用下划线。
2. 同属于一个文件夹/包名中的文件，也需要申明 import 才能引用。
3. 包申明中的两个关键字需要区分：
   * package：`.proto` 文件内部的范围定义，与具体的语言无关。
   * go\_package：如无提供 `option go_package`，将使用 `package` 定义的包名作为 Go 的包名。`option go_package` 需要提供完整的包名（相对于该项目的完整包路径，以 `go.mod` 文件中的 `module` 名为前缀）
4. 同包内的文件引用无需加包名前缀，跨包则需要。

## 参考

1. [Language Guide \(proto3\)](https://developers.google.com/protocol-buffers/docs/proto3#packages)
2. [Protobuf 的 import 功能在 Go 项目中的实践](https://studygolang.com/articles/25743)

