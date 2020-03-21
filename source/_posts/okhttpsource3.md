---
title: OkHttp源码分析（拓展）ConnectInterceptor CallServerInterceptor
date: 2020-03-21
tags: [Android,源码分析]
categories: 源码分析
toc: true
---

OkHttp源码分析（拓展）ConnectInterceptor CallServerInterceptor

<!--more-->

### ConnectInterceptor 链接拦截器


```java
public final class ConnectInterceptor implements Interceptor {
  public final OkHttpClient client;
  public ConnectInterceptor(OkHttpClient client) {
    this.client = client;
  }
  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    //读取了StreamAllocation 
    StreamAllocation streamAllocation = realChain.streamAllocation();
    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    //获取到我们使用的HttpCodec的实例 使用http 1.1版本协议 获取到的应该是Http1Codec实例
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    //获取到RealConnection对象实例
    RealConnection connection = streamAllocation.connection();
    //将获取到的这些业务实体对象传递给下一个拦截器
    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
}
```
接下来我们来看一下streamAllocation.newStream(client, chain, doExtensiveHealthChecks);


```java
public HttpCodec newStream(
    OkHttpClient client, Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
  //获取并设置这些超时时间
  int connectTimeout = chain.connectTimeoutMillis();
  int readTimeout = chain.readTimeoutMillis();
  int writeTimeout = chain.writeTimeoutMillis();
  int pingIntervalMillis = client.pingIntervalMillis();
  //是否开启链接失败后的重试逻辑
  boolean connectionRetryEnabled = client.retryOnConnectionFailure();
  try {
    //从ConnectionPool获取一个健康的链接 findHealthyConnection 调用了findConnection 最终调用了ConnectionPool里面的get方法
    RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
        writeTimeout, pingIntervalMillis, connectionRetryEnabled, doExtensiveHealthChecks);
    //根据我们获取到的链接创建了HttpCodec实例
    HttpCodec resultCodec = resultConnection.newCodec(client, chain, this);
    synchronized (connectionPool) {
      codec = resultCodec;
      return resultCodec;
    }
  } catch (IOException e) {
    throw new RouteException(e);
  }
}
```
CallServerInterceptor 链接拦截器

```java

@Override public Response intercept(Chain chain) throws IOException {
  RealInterceptorChain realChain = (RealInterceptorChain) chain;
  HttpCodec httpCodec = realChain.httpStream();
  StreamAllocation streamAllocation = realChain.streamAllocation();
  RealConnection connection = (RealConnection) realChain.connection();
  Request request = realChain.request();
  long sentRequestMillis = System.currentTimeMillis();
  realChain.eventListener().requestHeadersStart(realChain.call());
  //发起请求
  httpCodec.writeRequestHeaders(request);
  realChain.eventListener().requestHeadersEnd(realChain.call(), request);
  Response.Builder responseBuilder = null;
  if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
    // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
    // Continue" response before transmitting the request body. If we don't get that, return
    // what we did get (such as a 4xx response) without ever transmitting the request body.
    if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
      httpCodec.flushRequest();
      realChain.eventListener().responseHeadersStart(realChain.call());
      //获取响应
      responseBuilder = httpCodec.readResponseHeaders(true);
    }
    if (responseBuilder == null) {
      // Write the request body if the "Expect: 100-continue" expectation was met.
      realChain.eventListener().requestBodyStart(realChain.call());
      long contentLength = request.body().contentLength();
      CountingSink requestBodyOut =
          new CountingSink(httpCodec.createRequestBody(request, contentLength));
      BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);
      request.body().writeTo(bufferedRequestBody);
      bufferedRequestBody.close();
      realChain.eventListener()
          .requestBodyEnd(realChain.call(), requestBodyOut.successfulCount);
    } else if (!connection.isMultiplexed()) {
      // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
      // from being reused. Otherwise we're still obligated to transmit the request body to
      // leave the connection in a consistent state.
      streamAllocation.noNewStreams();
    }
  }
  httpCodec.finishRequest();
  if (responseBuilder == null) {
    realChain.eventListener().responseHeadersStart(realChain.call());
    responseBuilder = httpCodec.readResponseHeaders(false);
  }
  Response response = responseBuilder
      .request(request)
      .handshake(streamAllocation.connection().handshake())
      .sentRequestAtMillis(sentRequestMillis)
      .receivedResponseAtMillis(System.currentTimeMillis())
      .build();
  int code = response.code();
  if (code == 100) {
    // server sent a 100-continue even though we did not request one.
    // try again to read the actual response
    responseBuilder = httpCodec.readResponseHeaders(false);
    response = responseBuilder
            .request(request)
            .handshake(streamAllocation.connection().handshake())
            .sentRequestAtMillis(sentRequestMillis)
            .receivedResponseAtMillis(System.currentTimeMillis())
            .build();
    code = response.code();
  }
  realChain.eventListener()
          .responseHeadersEnd(realChain.call(), response);
  if (forWebSocket && code == 101) {
    // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
    response = response.newBuilder()
        .body(Util.EMPTY_RESPONSE)
        .build();
  } else {
    response = response.newBuilder()
        .body(httpCodec.openResponseBody(response))
        .build();
  }
  if ("close".equalsIgnoreCase(response.request().header("Connection"))
      || "close".equalsIgnoreCase(response.header("Connection"))) {
    streamAllocation.noNewStreams();
  }
  if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
    throw new ProtocolException(
        "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
  }
  //最终获取响应数据
  return response;
}

```

关于OkHttp的连接池及连接创建过程可以参见：
[https://sq.163yun.com/blog/article/188729834576564224](https://sq.163yun.com/blog/article/188729834576564224)