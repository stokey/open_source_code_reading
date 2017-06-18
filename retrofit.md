# Retrofit框架阅读
## 目标
+ [ ] 学会使用该框架
+ [ ] 了解核心技术点，及基本设计思想

## 阅读路径
1. 基本使用入手
2. 找出核心类
3. 根据核心类，绘制流程图
4. 根据流程图绘制核心类类图
5. 根据核心类类图和流程图分析框架设计思想
6. 详细解析某些功能点

### 基本使用方式

```java
public interface GithubService{
	@GET("user/{user}/repos")
	Call<List<Repo>> listRepos(@Path("user") String user);
}
Retrofit retrofit = new Retrofit.Builder()
	.baseUrl("http://api.github.com/")
	.build();
GithubService service = retrofit.create(GithubService.class);
Call<List<Repo>> repos = service.listRepos("octocat); // 发起请求，获取响应
```
### 核心类
+ Retrofit
	+ 相关类 
		+ Retrofit.Builder
		+ ServiceMethod
		+ OkHttpCall
	+ 核心方法
### 流程图
### 核心类类图
### 其他

## 深入分析