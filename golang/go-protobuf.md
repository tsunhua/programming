# Go 中使用 ProtoBuf

## package & import

1. package 名应该与所在文件夹同名，且只使用小写字母，不用大写字母也不用下划线。
2. 同属于一个文件夹/包名中的文件，也需要申明 import 才能引用。
3. 包申明中的两个关键字需要区分：
   * package：`.proto` 文件内部的范围定义，与具体的语言无关。
   * go\_package：如无提供 `option go_package`，将使用 `package` 定义的包名作为 Go 的包名。`option go_package` 需要提供完整的包名（相对于该项目的完整包路径，以 `go.mod` 文件中的 `module` 名为前缀）
4. 同包内的文件引用无需加包名前缀，跨包则需要。

## 问题

### 生成的字段如何注入 bson 标签

使用 protoc-go-inject-tag 可以解决该问题。操作如下：

（1）安装 `protoc-go-inject-tag`

```bash
go get github.com/favadi/protoc-go-inject-tag
```

（2）在 .proto 中使用 @inject\_tag 注释

```bash
message User {
    string id = 1;
    // @inject_tag: bson:"companyId"
    string companyId = 3;
}
```

（3）运行 `protoc` 进行编译后对生成的文件执行以下命令

```bash
protoc-go-inject-tag -input=/for/bar.pb.go
```

### 移除 proto3 默认生成的 XXX\_NoUnkeyedLiteral、XXX\_unrecognized、XXX\_sizecache 字段

使用 gogoprotobuf 可以解决该问题。具体操作如下：

（1）安装 `protoc-gen-gogofaster`

```bash
go get github.com/gogo/protobuf/protoc-gen-gofast
go get github.com/gogo/protobuf/proto
go get github.com/gogo/protobuf/protoc-gen-gogofaster
go get github.com/gogo/protobuf/gogoproto
```

（2）修改编译参数，由 `--go_out` 改为 `--gogofaster_out`

```bash
protoc --proto_path=. --proto_path=..  --gogofaster_out=plugins=grpc,paths=source_relative:${PROTOCOL_DIR} *.proto
```

## 参考

1. [Language Guide \(proto3\)](https://developers.google.com/protocol-buffers/docs/proto3#packages)
2. [Protobuf 的 import 功能在 Go 项目中的实践](https://studygolang.com/articles/25743)

