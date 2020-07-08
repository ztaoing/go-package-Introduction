# grpc
gRPC是出色的，现代的IDL和微服务传输。 如果您要启动一个未开发的项目，则go-kit强烈建议将gRPC作为默认传输方式。

一个重要的注意事项是，尽管gRPC支持流请求和答复，但是go-kit不支持。 您仍然可以在服务中使用流，但是其实现将无法利用中间件之类的许多go-kit功能。

一起使用gRPC和go-kit非常简单。

首先，使用protobuf3定义服务。 gRPC文档中对此进行了说明。 有关示例，请参见add.proto。 确保原型定义与您的服务的go-kit（接口）定义相匹配。

接下来，获取协议编译器。

您可以从protobuf发行页面下载预编译的二进制文件。 您将使用包含可执行文件的子目录bin解压缩名为protoc3的文件夹。 将该可执行文件移到$ PATH中的某个位置，您就可以开始了！
也可以从源代码构建。

```golang
brew install autoconf automake libtool
git clone https://github.com/google/protobuf
cd protobuf
./autogen.sh ; ./configure ; make ; make install
```

然后，编译服务定义，从.proto到.go。

```golang
protoc add.proto --go_out=plugins=grpc:.
```
最后，编写一个从您的服务定义到gRPC定义的微小绑定。 这是从一个域到另一个域的简单转换。 有关示例，请参见grpc_binding.go。

而已！ gRPC绑定可以绑定到侦听器，并处理正常的gRPC请求。 在您的服务中，您可以使用标准的go-kit组件和习惯用语。 有关gRPC支持的完整工作示例，请参见addsvc。 请记住：Go-kit服务可以同时支持多种传输。