# Volley源码解析

## 目标
+ [ ] 了解功能，学会用法
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
	+ 核心方法/属性 
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
		+ DiskBaseCache类解析 
	+ Network类解析
	+ HttpStack类解析
		+ HurlStack类解析
		+ HttpClientStack类解析 
		
## 深入解析
