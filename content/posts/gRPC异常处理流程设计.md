---
title: "gRPC异常处理流程设计"
date: 2021-04-18 13:53:26.861
draft: false
type: "post"
showTableOfContents: true
tags: ["gRPC"]
---

> 2021-06-30更新，新增了部分代码演示，请[参考](https://github.com/Raremaa/grpc-common)
> 2022-02-17更新，**之前思路有局限，这里不推荐使用异常来处理业务上的异常，基于此认知，更推荐业务上能够约定固定的数据结构ResultDTO，能够描述业务上的异常，比如错误码等信息，”异常“的处理，更推荐是在不可预知的非义务异常上进行兜底**，目前已在todo list中，准备重新设计，来补足业务上已知的异常处理方式。

# 1. 核心诉求

- 服务提供方异常能够被服务消费方感知
- 异常分类处理：
    1. 业务异常，需要返回对应的错误码，方便服务消费方进行国际化文案的提示+日志。
    2. 非业务异常(比如NPE)，需要返回内容给到服务消费方感知。
- 拓展性&流程尽可能简单

# 2. 方案选择

## 2.1. 直接调用OnError方法，传递Status包装异常后返回

例子：

```java
try {

}catch (Throwable t) {
// Throwable t ｜ StreamObserver<xxx> responseObserver
responseObserver.onError(Status.UNKNOWN
                  .withDescription(t.getMessage())
                  .withCause(t)
                  .asRuntimeException());
}
```

这个方式客户端可以感知到，但是可能能够放入的信息有限，只能是一个字符串，只能在`withDescription`这个参数里，如果要多个参数，可能借助一些序列化框架转化为字符串进行转换。

## 2.2. 借助protobuf 的 OneOf语法

protobuf文件：

```java
message Request {
  int32 number = 1;
}

message SuccessResponse {
  int32 result = 1;
}

enum ErrorCode {
  ABOVE_20 = 0;
  BELOW_2 = 1;
}

message ErrorResponse {
  int32 input = 1;
  ErrorCode error_code = 2;
}

// 重点是这里，回调有两种，一个成功，一个失败
message Response {
  oneof response {
    SuccessResponse success_response = 1;
    ErrorResponse error_response = 2;
  }
}

service CalculatorService {
  rpc findSquare(Request) returns (Response) {};
}
```

Java代码：

```java
@Override
public void findSquare(Request request, StreamObserver<Response> responseObserver) {
    Response.Builder builder = Response.newBuilder();
		try {
			// 异常业务
		}catch (Throwable t) {
				// 有异常的话返回错误的Message类型
        ErrorResponse errorResponse = ErrorResponse.newBuilder()
								// 业务异常
                .setInput(xxx)
								// 业务错误代码
                .setErrorCode(errorCode)
                .build();
        builder.setErrorResponse(errorResponse);
				return;
		}
		
		// 成功的话返回正确的Message类型
    builder.setSuccessResponse(SuccessResponse.newBuilder()).build());
    responseObserver.onNext(builder.build());
    responseObserver.onCompleted();
}
```

这种方式可以存放多个数据(只要在成功/失败的Message里定义更多字段)，但是需要定义两个Message(成功，失败各一个)才能完成业务，比较麻烦。

## 2.3. 基于gRPC Metadata(实际采用的方案)

protobuf文件：

这个protobuf文件建议反馈到基础架构部门集成到公司二方包中，避免每个项目再去定义。

```java
syntax = "proto3";

package credit ;
option java_package = "com.maycur.grpc.credit";

	// 通用异常处理信息
message ErrorInfo {

  // 错误的业务编码
  string errorCode  = 1;

  // 默认提示信息
  string defaultMsg = 2;
}
```

java代码：

```java
try{
	// 业务代码 | StreamObserver<xxx> responseObserver
} catch(Throwable t) {
	if (t instanceof ServiceException) {
            // 业务异常，返回错误码和默认文案到客户端
            ServiceException serviceException = (ServiceException) t;
            Metadata trailers = new Metadata();
            Error.ErrorInfo errorInfo = Error.ErrorInfo.newBuilder()
                    .setErrorCode(serviceException.getErrorCode())
                    .setDefaultMsg(serviceException.getMessage())
                    .build();
            Metadata.Key<Error.ErrorInfo> ERROR_INFO_TRAILER_KEY =
                    ProtoUtils.keyForProto(errorInfo);
            trailers.put(ERROR_INFO_TRAILER_KEY, errorInfo);
            responseObserver.onError(Status.UNKNOWN
                    .withCause(serviceException)
                    .asRuntimeException(trailers));
        } else {
            // 非业务异常，返回异常详情到客户端。
            responseObserver.onError(Status.UNKNOWN
                    // 这里就是我们的自定义异常信息
                    .withDescription(t.getMessage())
                    .withCause(t)
                    .asRuntimeException());
        }
        // 抛出异常让上层业务感知(比如事务回滚等可能要用到)
        throw new RuntimeException(t);
}
```

总的来看就是这个图：

![gRPC%E5%BC%82%E5%B8%B8%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B%E8%AE%BE%E8%AE%A1%208a3a50f6c0094091ab8fb55dd1532768/Untitled.png](https://img.masaiqi.com/20210418135241.png)

## 2.4. 优化的方案

上面的方案有一点不够优雅，因为所有的异常都需要手动try-catch，非常的冗余。有几个方案优化他。

### 2.4.1. 实现gRPC提供的ServerInterceptor接口

```java
class ExceptionInterceptor implements ServerInterceptor {
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call, Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {
        ServerCall.Listener<ReqT> reqTListener = next.startCall(call, headers);
        return new ExceptionListener(reqTListener, call);
    }
}

class ExceptionListener extends ServerCall.Listener {
    ......
    public void onHalfClose() {
        try {
            this.delegate.onHalfClose();
        } catch (Throwable t) {
            // 统一处理异常
            ExtendedStatusRuntimeException exception = fromThrowable(t);
            // 调用 call.close() 发送 Status 和 metadata
            // 这个方式和 onError()本质是一样的
            call.close(exception.getStatus(), exception.getTrailers());
        }
    }
}
```

这个方案核心其实就是不调用`onError()`方法，直接手动执行onError方法里面的内容，但是个人认为这样做违背编程契约，如果后续gRPC代码改动，这里有潜在风险。

### 2.4.2. 包装StreamObserver类，增强其功能。(采用的方案)

```java
/**
 * gRPC回调委派(装饰)类，职责是增强原有的{@link StreamObserver}, 新增捕获gRPC的异常并执行相应的处理
 * <p>
 * {@link GrpcService} 应该组合这个类，通过该类进行委派实现
 * <p>
 * Thread-unSafe implementation
 *
 * @author masaiqi
 * @date 2021/4/12 18:11
 */
public class StreamObserverDelegate<ReqT extends Message, RespT extends Message> implements StreamObserver<RespT> {

    private static final Logger logger = LoggerFactory.getLogger(StreamObserverDelegate.class);

    private StreamObserver<RespT> originResponseObserver;

    public StreamObserverDelegate(StreamObserver<RespT> originResponseObserver) {
        Assert.notNull(originResponseObserver, "originResponseObserver must not null!");
        this.originResponseObserver = originResponseObserver;
    }

    @Override
    public void onNext(RespT value) {
        this.originResponseObserver.onNext(value);
    }

    @Override
    public void onError(Throwable t) {
        if (t instanceof ServiceException) {
            // 业务异常，返回错误码和默认文案到客户端
            ServiceException serviceException = (ServiceException) t;
            Metadata trailers = new Metadata();
            Error.ErrorInfo errorInfo = Error.ErrorInfo.newBuilder()
                    .setErrorCode(serviceException.getErrorCode())
                    .setDefaultMsg(serviceException.getMessage())
                    .build();
            Metadata.Key<Error.ErrorInfo> ERROR_INFO_TRAILER_KEY =
                    ProtoUtils.keyForProto(errorInfo);
            trailers.put(ERROR_INFO_TRAILER_KEY, errorInfo);
            this.originResponseObserver.onError(Status.UNKNOWN
                    .withCause(serviceException)
                    .asRuntimeException(trailers));
        } else {
            // 非业务异常，返回异常详情到客户端。
            this.originResponseObserver.onError(Status.UNKNOWN
                    // 这里就是我们的自定义异常信息
                    .withDescription(t.getMessage())
                    .withCause(t)
                    .asRuntimeException());
        }
        // 抛出异常让上层业务感知(比如事务回滚等可能要用到)
        throw new RuntimeException(t);
    }

    @Override
    public void onCompleted() {
        if (originResponseObserver != null) {
            originResponseObserver.onCompleted();
        }
    }

    /**
     * 执行业务(自动处理异常)
     *
     * @author masaiqi
     * @date 2021/4/12 18:11
     */
    public RespT executeWithException(Function<ReqT, RespT> function, ReqT request) {
        RespT response = null;
        try {
            response = function.apply(request);
        } catch (Throwable e) {
            this.onError(e);
        }
        return response;
    }

    /**
     * 执行业务(自动处理异常)
     *
     * @author masaiqi
     * @date 2021/4/12 18:11
     */
    public RespT executeWithException(Supplier<RespT> supplier) {
        RespT response = null;
        try {
            response = supplier.get();
        } catch (Throwable e) {
            this.onError(e);
        }
        return response;
    }
}
```

有了上面的委派类(包装类)后，就可以很方便的实现gRPC的方法了，以一个Unary RPC的调用方式为例：

```java
public class xxxGrpc extends xxxImplBase {

    @Override
    public void saveXXX(XXX request, StreamObserver<XXX> responseObserver) {
        StreamObserverDelegate streamObserverDelegate = new StreamObserverDelegate(responseObserver);
        streamObserverDelegate.executeWithException(() -> {
						// 业务代码
            return xxx;
        });
    }
}
```

服务提供方通过下面方式拿到异常信息：

![gRPC%E5%BC%82%E5%B8%B8%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B%E8%AE%BE%E8%AE%A1%208a3a50f6c0094091ab8fb55dd1532768/Untitled%201.png](https://img.masaiqi.com/20210418135242.png)

# 3. 参考文档

- [gRPC Error Handling](https://www.vinsguru.com/grpc-error-handling/)(有3种方案，可以参考下)
- [https://github.com/grpc/grpc-java/issues/2189](https://github.com/grpc/grpc-java/issues/2189)
- [https://skyao.gitbooks.io/learning-grpc/content/server/status/exception_process.html](https://skyao.gitbooks.io/learning-grpc/content/server/status/exception_process.html)
- [https://www.vinsguru.com/grpc-error-handling/](https://www.vinsguru.com/grpc-error-handling/)