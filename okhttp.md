# OkHttp框架阅读

## 目标
+ [ ] 搞懂框架实现的功能，及其核心技术点
+ [ ] 了解框架的设计的核心思想
+ [ ] 学会使用该框架

## 阅读路径
+ 先从用法入手，找出核心类及方法，画出流程图或者用户实例图
+ 然后根据特定功能进行深入阅读


## 源码解析
+ 框架简单用法

```java
OkHttpClient eagerClient = client.newBulider()
	.readTimeout(500,TimeUnit.MILLISECONDS)
	.build();
Response response = eagerClient.newCall(request).execute();
```
	
+ 分析出两个核心类：`OkHttpClient`,`Response`，一个设计模式：`建造者模式`
	+ OkHttpClinet
		+ 实现了三个接口——`Cloneable`,`Call.Factory`,`WebSocket.Factory` 
			+ 实现Cloneable接口用于实现clone方法实现快速复制对象
		+ 核心类
			+ Builder：`建造者模式`，用于初始化okhttpClient 【private static final class】
				+ Dispatcher
				+ Proxy
				+ List<Protocol>
				+ List<Interceptor> connectionSpecs
				+ List<Interceptor> networkInterceptors
				+ eventListenerFactory
				+ cookieJar
				+ Cache
				+ SocketFactory
				+ SSLSocketFactory
				+ ConnectionPool
				+ `followRedirects/followSslRedirects=true`
				+ `retryOnConnectionFailure=true`
				+ `connectionTimeout=10`
				+ `readTimeout/writeTimeout=10`
				+ `pingInterval=0` 
			+ Dispather
			+ RealCall
			+ Request
			+ Response:实现`Closeable`接口
				+ code
				+ message
				+ request
				+ body
				+ time[send/receive]
				+ network/cache/priorResponse 
		+ 核心方法
			+ newBuilder()->okhttp.Builder
			+ newCall(Request request)->Call 
			+ newWebSocket(Request request, WebSocketListener listener)->RealWeSocket
	+ RealCall 
		+ 核心类
			+ Dispatcher：管理任务队列
			+ AsyncCall：final class(内部类)，处理异步请求 
			+ RealInterceptorChain：获取Reponse
			+ Response：请求响应类
		+ 核心方法
			+ execute()->Response: 同步方法，执行请求,` Dispatcher.executed()->getResponseWithInterceptorChain()->Dispatcher.finish()` 
			+ enqueue(CallBack callBack)->void: 异步方法加入队列,`Dispatcher.enqueue(new AsyncCall(callBack))`
			+  getResponseWithInterceptorChain()->Response
				+ List<Interceptor> interceptors
				+ add client.interceptors()  
				+ add retryAndFollowUpInterceptor
				+ add BridgeInterceptor
				+ add CacheInterceptor
				+ add ConnectInterceptor
				+ !isWebsocket add client.networkInterceptors()
				+ add CallServerInterceptor
				+ initial `Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0, originalRequest);`
				+ Interceptor.Chain.proceed(originalRequest)
	+ AsyncCall：RealCall的内部类
		+ 核心成员
			+ Callback responseCallback
				+ onFailure()
				+ onResponse() 
		+ 核心方法
			+ execute()->void: `getResponseWithInterceptorChain() -> responseCallback.onFailure()/responseCallback.onResponse() -> Dispatcher.finish()`    
	+ Dispatcher：管理任务队列
		+ 核心类
			+ Interceptor
			+ Interceptor。Chain	  
		+ 核心成员
			+ maxRequests = 64
			+ maxRequestsPerHost = 5
			+ Runnable idleCallback
			+ ExecutorService
			+ Deque<AsyncCall> readyAsyncCalls
			+ Deque<AsyncCall> runningAsyncCalls
			+ Deque<AsyncCall> runningSyncCalls 
		+ 核心方法
			+ executorService()->ExecutorService
			+ setMaxRequests/MaxRequestsPerHost/IdleCallback
			+ getMaxRequests/MaxRequestsPerHost/IdleCallback
			+ queued/runningCalls/CallsCount
			+ executed(RealCall)->void: `runningSyncCalls.add(call);`
			+ enqueue(AsybcCall)->void

			```java
			if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
				runningAsyncCalls.add(call);
				executorService().execute(call);
			} else {
				readyAsyncCalls.add(call);
			}
    		```
    + RealInterceptorChain：管理请求的返回结果		