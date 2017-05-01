# Volley源码解析

## 目标
+ [X] 了解功能，学会用法
+ [X] 了解框架设计核心思想及相关技术点  

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
+ ResponseDelivery类
	+ ExecutorDelivery
		+ ImmediateResponseDelivery
	+ MockResponseDelivery  
+ Network类
	+ BasicNetwork
	+ MockNetwork
+ HttpStack类
	+ HttpClientStack
	+ HurlStack
	+ MockHttpStack  
 
## 流程图
![img](./images/volley/volley_flow_chart.png)
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
		+ 核心属性
			+ DEFAULT_CACHE_DIR：`磁盘缓存默认文件夹`
		+ 核心方法
			+ newRequestQueue(Context,HttpStack)->RequestQueue
				+ 创建默认RequestQueue工作池/缓存目录
				+ 决定底层Network方式：`Android 2.3及其以上采用HurlStack方式通信——HttpURLConnection，以下采用HttpClientStack方式通信——HttpClient`
				
					```java
					// 此处可以通过自定义HttpStack决定，底层http客户端通信方式——volley+okhttp
					if (stack == null) {
						if (Build.VERSION.SDK_INT >= 9) {
							// HttpClient
							stack = new HurlStack();
						} else{
							// HttpUrlConnection
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
			+ DEFAULT_ NETWORK _ THREAD _ POOL _ SIZE：网络请求线程池大小，默认值为4
			+ 两种处理机制
				+ Cache：缓存机制
					+ Cache cache：通过缓存文件处理响应
					+ PriorityBlockingQueue<Request<?>> mCacheQueue 
					+ CacheDispatcher mCacheDispatcher
				+ Network
					+ Network mNetwork：通过请求网络处理响应
					+ PriorityBlockingQueue<Request<?>> mNetworkQueue 
					+ NetworkDispatcher[] mDispatchers
			+ start()方法：启动队列中的dispatchers[dispatcher会调用start方法]
			
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
			+ performRequest(Request<?> request)：`执行请求方法，返回NetworkResponse`
		+ 实现类
			+ BasicNetwork类解析：`处理Http网络请求类`
				+  核心属性／方法说明
					+ 核心属性  
						+ DEFAULT_POOL_SIZE：`默认缓存池大小，默认值为4096 byte`	  
						+ SLOW_REQUEST_THRESHOLD_MS：`网络请求日志打印的间隔时间，默认3000 ms`
						+ HttpStack mHttpStack：`Volley类带过来的HttpStack client，最终执行网络请求的类`
						+ ByteArrayPool mPool：缓存池大小
					+ 核心方法
						+ performRequest(Request<?> request)：`返回值NetworkResponse，HttpStack执行performRequest()方法获取HttpResponse,再将HttpResponse封装成NetworkResponse对象进行返回，或者调用重试方法进行重试，或者返回异常`
							+ 流程图
![img](./images/volley/volley_basic_network_flow_chart.png)
						
						+ addCacheHeaders(Map<String, String> headers, Cache.Entry entry)：`添加[If-None-Match/If-Modified-Since]字段到请求头`
						+  convertHeaders(Header[] headers)：`返回值为Map<String,String>类型，把响应头信息转换成Map<String,String>`
						+  entityToBytes(HttpEntity entity)：`放回值byte[]，将HttpEntity 转换成byte[]`
						+  logSlowRequests(long requestLifetime, Request<?> request,byte[] responseContents, StatusLine statusLine)：`打印请求信息`
						+  attemptRetryOnException(String logPrefix, Request<?> request,VolleyError exception)：`尝试重试请求——request.addMarker，performRequest()方法一直轮询，如果不返回或者异常则会一直请求`
							+ BasicNetwork异常处理机制流程图
![img](./images/volley/volley_basic_network_exception_flow_chart.png)  
			+ MockNetwork类解析：`用于Http网络请求测试类` 
	+ Dispatcher机制
		+ 类图说明
![img](./images/volley/volley_dispatcher_uml.png) 
		+ 实现类
			+ NetworkDispatcher类解析
				+ 类说明：`处理网络请求类，Thread子类。在RequestQueue类中默认开辟四条线程，同时处理mNetworkQueue中的请求队列` 
				+ 核心属性／方法说明 
					+ 核心属性
						+ BlockingQueue<Request<?>> mQueue：`网络请求队列，RequestQueue类中start()方法传入` 
						+ Cache mCache：`用于写入和读取缓存的类`
						+ ResponseDelivery mDelivery：`用于处理响应的类`
						+ boolean mQuit：`判断是否结束的标志，默认值false`
						+ Network mNetwork：`实际处理http请求的实体类`
					+ 核心方法 
						+ parseAndDeliverNetworkError()：`将异常封装后作为响应结果返回`
						+ quit()：`结束操作，设置mQuit=true同时调用Thread.interrupt()方法退出线程`
						+ run()：`处理网络请求，获取响应，处理响应缓存` 
							+ 流程图
![img](./images/volley/volley_network_dispatcher_run_flow_chart.png)
			+ CacheDispatcher类解析
				+ 类说明：`处理响应请求缓存类，Thread子类，在RequestQueue类中的start()方法中调用CacheDispatcher的start()方法执行轮询操作` 
				+ 核心属性／方法说明 
					+ 核心属性
						+ BlockingQueue<Request<?>> mCacheQueue：`请求的缓存队列`
						+ BlockingQueue<Request<?>> mNetworkQueue：`网络请求队列，和NetworkDispatcher共享，RequestQueue类中start()方法传入` 
						+ Cache mCache：`用于写入和读取缓存的类`
						+ ResponseDelivery mDelivery：`用于处理响应的类`
						+ boolean mQuit：`判断是否结束的标志，默认值false`
					+  核心方法
						+ quit()：`结束操作，设置mQuit=true同时调用Thread.interrupt()方法退出线程`
						+ run()：`处理请求，读取缓存，写入缓存`
							+ 流程图 
![img](./images/volley/volley_cache_dispatcher_run_flow_chart.png)
	+ HttpStack类解析
		+ 类图说明
![img](./images/volley/volley_httpstack_uml.png)
		+ 核心方法说明
			+ performRequest(Request<?> request, Map<String, String> additionalHeaders)->HttpResponse：`发起请求，获取响应的抽象方法` 
		+ 实现类 
			+ MockHttpStack类解析：`用于测试的测试类`
			+ HurlStack类解析：`底层封装的是HttpURLConnection，适用于Android 2.3及其以上版本`
			+ HttpClientStack类解析：`底层封装的是HttpClient，适用于Android 2.3以下版本`
	+ Response类解析：`响应范型类`
		+ 类图说明
![img](./images/volley/volley_response_uml.png)
		+ 核心属性／方法说明
			+ 核心属性
				+ T result：`响应结果，范型类型，可能是String/Json/Bitmap`
				+ Cache.Entry cacheEntry：`用于处理响应的缓存对象`
				+ VolleyError error：`响应异常信息，如果响应正常则为null`
				+ boolean intermediate：`如果设为true，第一次响应默认过期响应，第二次响应会触发返回。相当于返回的响应请求过期，然后调动二次请求进行响应`
			+ 核心方法 
				+ isSuccess()：`判断响应是否成功，error == null`
				+ success(T result,Cache.Entry cacheEntry)->Respnse<T>：`把响应结果T result封装成Response<T>对象进行返回` 
				+ error(Volley error)->Response<T>：`将响应错误封装成Response<T>对象进行返回`
				
	+ ResponseDelivery类解析：`将响应传递出去的接口`
		+ 类图说明
![img](./images/volley/volley_response_delivery_uml.png)
		+ 核心方法说明
			+ postResponse()：`将响应信息发送给请求`
			+ postError()：`将错误信息发送给请求` 
		+ 实现类
			+ ExecutorDelivery
				+ 内部类
					+ ResponseDeliveryRunnable：`Runnable子类，用于异步处理响应信息`
					+ 核心方法解析
						+ run()方法流程图
![img](./images/volley/volley_response_delivery_runable_run_flow_chart.png)   
				+ 核心属性／方法说明
					+ 核心属性
						+ Executor mResponsePoster：`处理响应信息的执行器，最终通过传递过来的handler通过post()方法将ResponseDeliveryRunnable对象传递出去`  
					+ 核心方法
						+ postResponse()：`传递响应结果方法`
						+ postError()：`传递响应错误方法`
				+ Response消息传递流程图
![img](./images/volley/volley_response_flow_chart.png) 
				+ 继承类 
					+ ImmediateResponseDelivery
					
					```java
					public ImmediateResponseDelivery() {
						super(new Executor() {
							@Override
							public void execute(Runnable command) {
								command.run();
							}
						});
					}
					```  
			+ MockResponseDelivery：`用于测试，MOCK HTTP`
	+ Request类解析
		+ 类图说明
![img](./images/volley/volley_request_uml.png)
		+ 核心属性／方法说明
		+ 实现类
			+ StringRequest
				+ 核心属性／方法说明
					+ 核心属性
						+ Listener<String> mListener：`用于监听正确响应的监听器` 
					+ 核心方法
						+ deliverResponse()：`调用监听的响应方法，实现回调`

						```java
						@Override
						protected void deliverResponse(String response) {
							if (mListener != null) {
								mListener.onResponse(response);
							}
						}
						```
						
						+ parseNetworkResponse()：`将NetworkResponse转换成Response<String>对象` 
						
						```java
						@Override
						protected Response<String> parseNetworkResponse(NetworkResponse response) {
							String parsed;
							try {
								parsed = new String(response.data,HttpHeaderParser.parseCharset(response.headers));
							} catch (UnsupportedEncodingException e) {
								parsed = new String(response.data);
							}
							return Response.success(parsed, HttpHeaderParser.parseCacheHeaders(response));
						}
						```
						
			+ JsonRequest<T>
				+ 类图说明
![img](./images/volley/volley_json_request_uml.png)
				+ 核心属性／核心方法说明
				+ 继承类 
					+ JsonObjectRequest
						+ 核心方法说明
							+ parseNetworkResponse()：`将NetworkResponse转换成Response<JSONObject对象`
							
							```java
							@Override
							protected Response<JSONObject> parseNetworkResponse(NetworkResponse response) {
							try {
								String jsonString = new String(response.data,HttpHeaderParser.parseCharset(response.headers, PROTOCOL_CHARSET));
								return Response.success(new JSONObject(jsonString),HttpHeaderParser.parseCacheHeaders(response));
							} catch (UnsupportedEncodingException e) {
									return Response.error(new ParseError(e));
							} catch (JSONException je) {
									return Response.error(new ParseError(je));
								}
							}
							```
					
					+ JsonArrayRequest
						+ 核心方法说明
							+ parseNetworkResponse()：`将NetworkResponse转换成Response<JSONArray>对象`
							
							```java
							@Override
							protected Response<JSONArray> parseNetworkResponse(NetworkResponse response) {
								try {
									String jsonString = new String(response.data,HttpHeaderParser.parseCharset(response.headers, PROTOCOL_CHARSET));
									return Response.success(new JSONArray(jsonString),HttpHeaderParser.parseCacheHeaders(response));
								} catch (UnsupportedEncodingException e) {
									return Response.error(new ParseError(e));
								} catch (JSONException je) {
									return Response.error(new ParseError(je));
								}
							}
							```
							
			+ ImageRequest
				+ 核心属性／方法
					+ 核心属性
					+ 核心方法
						+ doParse()：`将NetworkResponse转换成Response<Bitmap>对象`
						
						```java
						private Response<Bitmap> doParse(NetworkResponse response) {
							byte[] data = response.data;
							BitmapFactory.Options decodeOptions = new BitmapFactory.Options();
							Bitmap bitmap = null;
							if (mMaxWidth == 0 && mMaxHeight == 0) {
								decodeOptions.inPreferredConfig = mDecodeConfig;
								bitmap = BitmapFactory.decodeByteArray(data, 0, data.length, decodeOptions);
							} else {
								 // If we have to resize this image, first get the natural bounds.
								decodeOptions.inJustDecodeBounds = true;
								BitmapFactory.decodeByteArray(data,0,data.length,decodeOptions);
								int actualWidth = decodeOptions.outWidth;
								int actualHeight = decodeOptions.outHeight;
								// Then compute the dimensions we would ideally like to decode to.
								int desiredWidth = getResizedDimension(mMaxWidth, mMaxHeight,actualWidth, actualHeight, mScaleType);
								int desiredHeight = getResizedDimension(mMaxHeight, mMaxWidth,actualHeight, actualWidth, mScaleType);
								// Decode to the nearest power of two scaling factor.
								decodeOptions.inJustDecodeBounds = false;
								// TODO(ficus): Do we need this or is it okay since API 8 doesn't support it?
								// decodeOptions.inPreferQualityOverSpeed = PREFER_QUALITY_OVER_SPEED;
								decodeOptions.inSampleSize =findBestSampleSize(actualWidth, actualHeight, desiredWidth, desiredHeight);
								Bitmap tempBitmap = BitmapFactory.decodeByteArray(data, 0, data.length, decodeOptions);
								// If necessary, scale down to the maximal acceptable size.
								if (tempBitmap != null && (tempBitmap.getWidth() > desiredWidth ||tempBitmap.getHeight() > desiredHeight)) {
									bitmap = Bitmap.createScaledBitmap(tempBitmap,desiredWidth, desiredHeight, true);
									tempBitmap.recycle();
								} else {
									bitmap = tempBitmap;
								}
							}
							if (bitmap == null) {
								return Response.error(new ParseError(response));
							} else {
								return Response.success(bitmap, HttpHeaderParser.parseCacheHeaders(response));
							}
						}
						```
						   
						+ deliverResponse()
						
						```java
						@Override
						protected void deliverResponse(Bitmap response) {
							mListener.onResponse(response);
						}
						```

## 深入解析
+ Volley范型机制
	+ 范型类 
		+ Request<T>
			+ JsonRequest<T>
				+ JsonObjectRequest(`extends JsonRequest<JSONObject>`) 
				+ JsonArrayRequest(`extends JsonRequest<JSONArray>`) 
			+ ImageRequest(`extends Request<Bitmap>`)
			+ StringRequest(`extends Request<String>`) 
		+ Response<T> 
+ HttpClient/HttpURLConnection封装技巧
+ Response消息传递机制
+ Cache机制
