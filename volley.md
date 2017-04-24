# Volley源码解析

## 目标
+ [X] 了解功能，学会用法
+ [ ] 了解框架设计核心思想及相关技术点  

## 阅读路径
1. 使用方式入手
2. 解析核心类
3.  掌握相关操作流程
4.  绘制类图和流程图
5.  从架构和设计思想方面分析
6.  详细解析某些功能点
 
### 使用方式
 + 建立请求队列
 + 创建请求，监听响应
 + 把请求加入队列
 
```java
 RequestQueue mQueue = Volley.newRequestQueue(conext);
 StringRequest stringRequest = new StringRequest(method,url,new Listener(){
 		@Override
 		public void onResponse(String response) {
 			Log.d("doJsonRequest:"+url,response);
 		}
 }, new ErrorListener(){
 		 @Override
 		 public void onErrorResponse(VolleyError error) {
 		 	Log.e("doJsonRequest","error:",error);
 		 }
 })；
```

### 核心类
+ Volley类 
+ RequestQueue类
+ Request类
	+ StringRequest
	+ JsonRequest
		+ JsonObjectRequest
		+ JsonArrayRequest
	+ ImageRequest
+ Response类    
 
## 流程图
## 详细解析
+ Volley类解析
	+ 类图说明
![img](./images/volley/volley_uml.png)
	+ 核心类
		+ RequestQueue
		+ Network
		+ HurlStack
		+ HttpClientStack
		+ DiskBasedCache 
	+ 核心方法／属性 
		+ 属性`DEFAULT_CACHE_DIR`：磁盘缓存默认文件夹
		+ 方法
			+ newRequestQueue(Context,HttpStack):RequestQueue
				+ 创建默认RequestQueue工作池/缓存目录
				+ 决定底层Network方式：`Android 2.3及其以上采用HurlStack方式通信——HttpURLConnection，以下采用HttpClientStack方式通信——HttpClient`
				
				```java
			// 此处可以通过自定义HttpStack决定，底层http客户端通信方式【volley+okhttp】
			if (stack == null) {
				if (Build.VERSION.SDK_INT >= 9) {
					stack = new HurlStack();} else {
                	// Prior to Gingerbread, HttpUrlConnection was unreliable.
                	// See: http://android-developers.blogspot.com/2011/09/androids-http-clients.html
                	stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
                	}
        }
        Network network = new BasicNetwork(stack);
        RequestQueue queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
        queue.start();
				```
	
	+ RequestQueue类解析
		+ 类图说明
![img](./images/volley/volley_request_queue_uml.png)
		
		+ 核心属性／方法说明
			+ DEFAULT_NETWORK_THREAD_POOL_SIZE：网络请求线程池大小，默认值为4
			+ 两种处理机制
				+ Cache：缓存机制
					+ Cache cache：通过缓存文件处理响应
					+ PriorityBlockingQueue<Request<?>> mCacheQueue 
					+ CacheDispatcher mCacheDispatcher
				+ Network
					+ Network mNetwork：通过请求网络处理响应
					+ PriorityBlockingQueue<Request<?>> mNetworkQueue 
					+ NetworkDispatcher[] mDispatchers
			+ start()方法：启动队列中的dispatchers。[dispatcher会调用start方法]
			
			```java
			public void start() {
				stop();  // Make sure any currently running dispatchers are stopped.
        			// Create the cache dispatcher and start it.
        			mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
        			mCacheDispatcher.start();
        			// Create network dispatchers (and corresponding threads) up to the pool size.
        			// mDispatchers  = new NetworkDispatcher[threadPoolSize];
        		for (int i = 0; i < mDispatchers.length; i++) {
        			NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,mCache, mDelivery);
        			mDispatchers[i] = networkDispatcher;
        			networkDispatcher.start();
        		}
    		}
			``` 
		
			+ stop()方法：关闭cache和network dispatchers[会调用dispatcher的quit方法]
			+ add()方法：把请求加入请求队列
			
			```java
			public <T> Request<T> add(Request<T> request) {
				// Tag the request as belonging to this queue and add it to the set of current requests.
				request.setRequestQueue(this);
				synchronized (mCurrentRequests) {
					mCurrentRequests.add(request);
				}
				// Process requests in the order they are added.
				request.setSequence(getSequenceNumber());
				request.addMarker("add-to-queue");
				// If the request is uncacheable, skip the cache queue and go straight to the network.
				if (!request.shouldCache()) {
					mNetworkQueue.add(request);
					return request;
				}
				// Insert request into stage if there's already a request with the same cache key in flight.
				synchronized (mWaitingRequests) {
					String cacheKey = request.getCacheKey();
					if (mWaitingRequests.containsKey(cacheKey)) {
						// There is already a request in flight. Queue up.
						Queue<Request<?>> stagedRequests = mWaitingRequests.get(cacheKey);
						if (stagedRequests == null) {
							stagedRequests = new LinkedList<Request<?>>();
						}
						stagedRequests.add(request);
						mWaitingRequests.put(cacheKey, stagedRequests);
						if (VolleyLog.DEBUG) {
							VolleyLog.v("Request for cacheKey=%s is in flight, putting on hold.", cacheKey);
						}
					} else {
						// Insert 'null' queue for this cacheKey, indicating there is now a request in flight.
						mWaitingRequests.put(cacheKey, null);
						mCacheQueue.add(request);
					}
					return request;
				}
			}
			```
		+ finish()方法：mCurrentRequests／mWaitingRequests移除已经执行完成的请求，添加的监听器—RequestFinishedListener调用onRequestFinished方法进行通知
		+ 相关类
			+ CacheDispatcher
			+ NetworkDispatcher 
			+ Cache
			+ Request
			+ ResponseDelivery
	+ Network类解析
		+ 类图说明
![img](./images/volley/volley_network_uml.png) 
		+ 核心属性／方法说明
	+ Dispatcher机制
		+ 类图说明
![img](./images/volley/volley_dispatcher_uml.png) 
		+ 实现类
			+ NetworkDispatcher类解析
			+ CacheDispatcher类解析  
	+ HttpStack类解析
		+ 类图说明
		+ 实现类 
			+ HurlStack类解析
			+ HttpClientStack类解析 
			+ MockHttpStack类解析
		
## 深入解析
