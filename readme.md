# Go Kit GRPC example
- How to Run
```
go run main.go
```

## Reference

1. [stringsvc1](http://gokit.io/examples/stringsvc.html#stringsvc1)

2. [go-kit 与 grpc 结合开发微服务](http://www.articlechain.cn/articles/2018/04/27/1524822459139)

- go-kit是什么这里就不介绍了, 就算写也不如网上的文章写的正规. 这里简单记录一下go-kit中提供的3个核心概念

1. Service
2. Endpoint
3. Transport

go-kit官方示例`stringsvc1`中对这三个概念表现的比较得...一般. 

`Service`即是我们的业务逻辑, 为了得到清晰的分层架构, `Service`独立与另外两者, 且划分地很明确.(只不过`stringsvc1`用了接口表示, 感觉多此一举, 当然, 在实际场景中为了测试或是其他原因, 使用接口声明可能是必要的, 但用在示例中就会有些繁杂了.)

`Endpoint`是项目内部微服务之间通信的接口, 每个endpoint相当于protobuf中一个rpc字段. 在`stringsvc1`中并没有内部子服务间的交互, 而是直接把`Service`暴露为http接口了, 所以这部分的作用没有体现出来. ta是单个微服务模块, 但也是一个完整的服务.

`Transport`是go-kit为我们提供的, 暴露给外部访问的入口. 有grpc, 也有restful类型. `stringsvc1`示例中为restful, 使用curl访问即可.

这里说一下, `stringsvc1`中把`Service`包装为`Endpoint`类型的部分, 也就是`makeXXXEndpoint`函数, 是不是和grpc中的服务端代码很像? 不过这里没有用protobuf, 而是直接定义了`uppercaseRequest`, `uppercaseResponse`这种结构体. 从`Transport`访问到`Service`过程中, 有参数与响应的转换操作. 

在我们的示例中, 把这部分用gprc + protobuf代替, 同时把客户端操作抽象成另一个子服务, 来体现`Endpoint`的作用.

------

## 关于`transport.NewServer()`构造出来的`handler`函数

在`stringsvc1`中类似如下

```go
	countHandler := httptransport.NewServer(
		makeCountEndpoint(svc),
		decodeCountRequest,
		encodeResponse,
	)
```

在参考文章2中的示例如下

```go
    bookListHandler := grpc_transport.NewServer(  
        makeGetBookListEndpoint(),
        decodeRequest,
        encodeResponse,
    )
```

其实`makeXXXEndpoint()`的结果已经很明确了, 就是单纯的grpc server的实现代码. 而不同的`transport`执行的`NewServer()`只是go-kit提供的, 为我们包装为不同的类型的公开接口的处理函数而已.

http的NewServer得到了不同路由的handler, 可以理解为'控制器(controller)', 而`grpc`的NewServer则得到了...咳, 好像跟单纯的grpc接口没什么不同啊.

因为你看, 在http的NewServer中, 对应的`encode`与`decode`函数需要从request的body中读取并反序列化, 再把response的结构体对象序列化; 而在grpc的NewServer中, `encode`与`decode`函数根本什么也没做, 直接透传的.

------

在我们的例子中, 假想了两个服务: 员工管理中心UserManager与部门管理中心DepartmentManager. 有一些初始数据...不要在意数据内容.

我们可以创建新部门, 并指定新部门的人员选择, DepartmentManager会调用UserManager服务的`AddUser`完成人员记录的创建.

另外可以进行人员的委派, 这会更改员工记录中的所属部门, 另外在部门管理中心服务中也会更新部门下的员工.

示例如下

查询

```
$ curl -X POST -d '{}' localhost:10000/user/list
{"List":[{"Name":"李彦宏","Company":"百度"},{"Name":"马云","Company":"阿里"},{"Name":"马化腾","Company":"腾讯"}]}

$ curl -X POST -d '{}' localhost:10000/department/list
{"List":[{"Name":"百度","Users":[{"Name":"李彦宏","Company":"百度"}]},{"Name":"阿里","Users":[{"Name":"马云","Company":"阿里"}]},{"Name":"腾讯","Users":[{"Name":"马化腾","Company":"腾讯"}]}]}
```

创建新部门

```
$ curl -X POST -d '{"name": "京东", "users": [{"name": "刘强东", "company": "京东"}]}' localhost:10000/department/create
{}
$ curl -X POST -d '{}' localhost:10000/user/list
{"List":[{"Name":"李彦宏","Company":"百度"},{"Name":"马云","Company":"阿里"},{"Name":"马化腾","Company":"腾讯"},{"Name":"刘强东","Company":"京东"}]}
$ curl -X POST -d '{}' localhost:10000/department/list
{"List":[{"Name":"百度","Users":[{"Name":"李彦宏","Company":"百度"}]},{"Name":"阿里","Users":[{"Name":"马云","Company":"阿里"}]},{"Name":"腾讯","Users":[{"Name":"马化腾","Company":"腾讯"}]},{"Name":"京东","Users":[{"Name":"刘强东","Company":"京东"}]}]}
```

人员委派

```
$ curl -X POST -d '{"name": "李彦宏", "company": "阿里"}' localhost:10000/user/dispatch
{}
$ curl -X POST -d '{}' localhost:10000/user/list
{"List":[{"Name":"李彦宏","Company":"阿里"},{"Name":"马云","Company":"阿里"},{"Name":"马化腾","Company":"腾讯"}]}
$ curl -X POST -d '{}' localhost:10000/department/list
{"List":[{"Name":"百度"},{"Name":"阿里","Users":[{"Name":"马云","Company":"阿里"},{"Name":"李彦宏","Company":"阿里"}]},{"Name":"腾讯","Users":[{"Name":"马化腾","Company":"腾讯"}]}]}```
```

`main.go`入口程序其实是微服务架构中的网关API服务, 在两个微服务中, 我们启动了grpc接口, 又注册了http路由, 由`main.go`接受用户访问, 然后分发.

当然实际场景中, 网关与后端微服务的连接应该是通过 restful 或是 grpc 的形式, 而不是这里"内嵌"的形式.
