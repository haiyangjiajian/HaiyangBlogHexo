---
layout: post
title: Android常用网络请求框架简析
tags: [android]
category: coder
---

# Android常用网络请求框架简析

# 目录
- 框架做什么
- 常用请求方式
- Retrofit简析

# 框架做什么

网络请求框架是为了方便我们的业务开发而设计的。如果脱离开网络框架可以想象一下Android程序如何发出一个网络请求。需要设置合适的参数（keep-alive，content-type，timeout，get/post等），然后创建一个socket，发出http请求，等待接收响应，解析数据，进行处理。Android还需要一个线程切换，启动一个线程发出请求，handler把结果发送到主线程。上述只是一个网络请求最简单的情况，可能还要涉及复杂的网络请求，上传图片，需要处理缓存，结果异常处理等。

这些都是和业务代码没有关系的网络连接的管理。网络请求框架就是帮我们封装这些细节的处理，使可以通过简单的配置就能进行网络请求。

框架要做的事情

- 异步请求
- 线程池
- 缓存
- 数据解析
- 错误处理
- 等等

<!-- more -->


一个好的网络请求框架应该可以和业务代码的**耦合性尽量降低**。

# 常用请求方式

### HttpClient

httpClient 在 Android 5.0 被废弃了，在 Android 6.0被删除。在官网中对其做了如下说明：“Android 6.0 release removes support for the Apache HTTP client. If your app is using this client and targets Android 2.3 (API level 9) or higher, use the HttpURLConnection class instead. This API is more efficient because it reduces network use through transparent compression and response caching, and minimizes power consumption.”

### HttpURLConnection
一个轻量级的 http 请求客户端，谷歌官方支持的，提供的 api 比较简单，比较容易使用和扩展。

### okhttp

[okhttp](https://github.com/square/okhttp)是一个高性能的http请求框架。支持同步get，异步get，post请求，提交文件，提交分块请求，可以响应缓存，取消一个请求，连接、读取、写入超时，处理验证等。okhttp底层流处理是基于okio，它补充了 java.io 和 java.nio 的内容，使得数据访问、存储和处理更加便捷。

### volley

volley是谷歌官方支持的。Android 版本大于9使用HttpURLConnection，小于9使用 HttpClient 进行通信。不支持 post 大数据，所以不适合上传文件。不过 Volley 设计的初衷本身也就是为频繁的、数据量小的网络请求而生

不过在现在17年10月份来看volley更新并不太活跃了

![include](/assets/img/2017-10-24_volley.png)


可以通过源码来分析一下volley的是如何将网络请求发送出去的。在其中的一些细节，可以让我门对volley的一些性质有更深入的了解。

volley需要调用Volley类中的静态方法newRequestQueue来生成一个requestQueue来管理执行的网络请求。

```java
public static RequestQueue newRequestQueue(Context context, HttpStack stack) {
        File cacheDir = new File(context.getCacheDir(), DEFAULT_CACHE_DIR);

        String userAgent = "volley/0";
        try {
            String packageName = context.getPackageName();
            PackageInfo info = context.getPackageManager().getPackageInfo(packageName, 0);
            userAgent = packageName + "/" + info.versionCode;
        } catch (NameNotFoundException e) {
        }

        if (stack == null) {
            if (Build.VERSION.SDK_INT >= 9) {
                stack = new HurlStack();
            } else {
                // Prior to Gingerbread, HttpUrlConnection was unreliable.
                // See: http://android-developers.blogspot.com/2011/09/androids-http-clients.html
                stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
            }
        }

        Network network = new BasicNetwork(stack);

        RequestQueue queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
        queue.start();

        return queue;
    }

```

可以看其中的 if (Build.VERSION.SDK_INT >= 9)这句起，是Android 版本大于9使用HttpURLConnection，小于9使用 HttpClient 进行通信的具体实现。创建的requestQueue有一个Network的对象。执行到最后是queue.start()。看一下具体实现。

```java
/**
     * Starts the dispatchers in this queue.
     */
    public void start() {
        stop();  // Make sure any currently running dispatchers are stopped.
        // Create the cache dispatcher and start it.
        mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
        mCacheDispatcher.start();

        // Create network dispatchers (and corresponding threads) up to the pool size.
        for (int i = 0; i < mDispatchers.length; i++) {
            NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,
                    mCache, mDelivery);
            mDispatchers[i] = networkDispatcher;
            networkDispatcher.start();
        }
    }
   
```

这段程序中启了一个 CacheDispatcher 线程用来处理缓存的请求，启动mDispatchers.length 个 NetworkDispatcher 用来处理实际发送的网络请求，默认是四个，NetworkDispatcher 有一个 Network对象是最后请求的实际发送者。这五个线程会一直在后台运行。**实际网络请求的线程数量有多个，且可以配置**是volley可以同时发送大量的请求的网络请求的原因之一。

执行的五个线程准备好了之后，等待requestQueue.add方法加入需要执行的request

```java
 /**
     * Adds a Request to the dispatch queue.
     * @param request The request to service
     * @return The passed-in request
     */
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
                // Insert 'null' queue for this cacheKey, indicating there is now a request in
                // flight.
                mWaitingRequests.put(cacheKey, null);
                mCacheQueue.add(request);
            }
            return request;
        }
    }
```

这段代码会判断当前请求配置的是否可以缓存，如果不可以缓存则直接加入到 mNetworkQueue 中，如果可以缓存则将请求加入到 mCacheQueue 。CacheDispatcher 会监听mCacheQueue。NetworkDispatcher会监听 mNetworkQueue。

在 CacheDispatcher 中会查找放入 mCacheQueue 的请求是否有缓存。如果缓存命中，返回结果，如果未命中放入 mNetworkQueue 中，看一下 NetworkDispatcher 的 run 方法

``` java
public void run() {
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        while (true) {
            long startTimeMs = SystemClock.elapsedRealtime();
            Request<?> request;
            try {
                // Take a request from the queue.
                request = mQueue.take();
            } catch (InterruptedException e) {
                // We may have been interrupted because it was time to quit.
                if (mQuit) {
                    return;
                }
                continue;
            }

            try {
                request.addMarker("network-queue-take");

                // If the request was cancelled already, do not perform the
                // network request.
                if (request.isCanceled()) {
                    request.finish("network-discard-cancelled");
                    continue;
                }

                addTrafficStatsTag(request);

                // Perform the network request.
                NetworkResponse networkResponse = mNetwork.performRequest(request);
                request.addMarker("network-http-complete");

                // If the server returned 304 AND we delivered a response already,
                // we're done -- don't deliver a second identical response.
                if (networkResponse.notModified && request.hasHadResponseDelivered()) {
                    request.finish("not-modified");
                    continue;
                }

                // Parse the response here on the worker thread.
                Response<?> response = request.parseNetworkResponse(networkResponse);
                request.addMarker("network-parse-complete");

                // Write to cache if applicable.
                // TODO: Only update cache metadata instead of entire record for 304s.
                if (request.shouldCache() && response.cacheEntry != null) {
                    mCache.put(request.getCacheKey(), response.cacheEntry);
                    request.addMarker("network-cache-written");
                }

                // Post the response back.
                request.markDelivered();
                mDelivery.postResponse(request, response);
            } catch (VolleyError volleyError) {
                volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                parseAndDeliverNetworkError(request, volleyError);
            } catch (Exception e) {
                VolleyLog.e(e, "Unhandled exception %s", e.toString());
                VolleyError volleyError = new VolleyError(e);
                volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                mDelivery.postError(request, volleyError);
            }
        }
    }

```

可以关注这句话 NetworkResponse networkResponse = mNetwork.performRequest(request); 这句执行了具体的网络请求。读一下它的代码，看看其对网络请求的发送如何处理。


```java
public NetworkResponse performRequest(Request<?> request) throws VolleyError {
        long requestStart = SystemClock.elapsedRealtime();
        while (true) {
            HttpResponse httpResponse = null;
            byte[] responseContents = null;
            Map<String, String> responseHeaders = Collections.emptyMap();
            try {
                // Gather headers.
                Map<String, String> headers = new HashMap<String, String>();
                addCacheHeaders(headers, request.getCacheEntry());
                httpResponse = mHttpStack.performRequest(request, headers);
                StatusLine statusLine = httpResponse.getStatusLine();
                int statusCode = statusLine.getStatusCode();

                responseHeaders = convertHeaders(httpResponse.getAllHeaders());
                // Handle cache validation.
                if (statusCode == HttpStatus.SC_NOT_MODIFIED) {

                    Entry entry = request.getCacheEntry();
                    if (entry == null) {
                        return new NetworkResponse(HttpStatus.SC_NOT_MODIFIED, null,
                                responseHeaders, true,
                                SystemClock.elapsedRealtime() - requestStart);
                    }

                    // A HTTP 304 response does not have all header fields. We
                    // have to use the header fields from the cache entry plus
                    // the new ones from the response.
                    // http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10.3.5
                    entry.responseHeaders.putAll(responseHeaders);
                    return new NetworkResponse(HttpStatus.SC_NOT_MODIFIED, entry.data,
                            entry.responseHeaders, true,
                            SystemClock.elapsedRealtime() - requestStart);
                }

                // Some responses such as 204s do not have content.  We must check.
                if (httpResponse.getEntity() != null) {
                  responseContents = entityToBytes(httpResponse.getEntity());
                } else {
                  // Add 0 byte response as a way of honestly representing a
                  // no-content request.
                  responseContents = new byte[0];
                }

                // if the request is slow, log it.
                long requestLifetime = SystemClock.elapsedRealtime() - requestStart;
                logSlowRequests(requestLifetime, request, responseContents, statusLine);

                if (statusCode < 200 || statusCode > 299) {
                    throw new IOException();
                }
                return new NetworkResponse(statusCode, responseContents, responseHeaders, false,
                        SystemClock.elapsedRealtime() - requestStart);
            } catch (SocketTimeoutException e) {
                attemptRetryOnException("socket", request, new TimeoutError());
            } catch (ConnectTimeoutException e) {
                attemptRetryOnException("connection", request, new TimeoutError());
            } catch (MalformedURLException e) {
                throw new RuntimeException("Bad URL " + request.getUrl(), e);
            } catch (IOException e) {
                int statusCode = 0;
                NetworkResponse networkResponse = null;
                if (httpResponse != null) {
                    statusCode = httpResponse.getStatusLine().getStatusCode();
                } else {
                    throw new NoConnectionError(e);
                }
                VolleyLog.e("Unexpected response code %d for %s", statusCode, request.getUrl());
                if (responseContents != null) {
                    networkResponse = new NetworkResponse(statusCode, responseContents,
                            responseHeaders, false, SystemClock.elapsedRealtime() - requestStart);
                    if (statusCode == HttpStatus.SC_UNAUTHORIZED ||
                            statusCode == HttpStatus.SC_FORBIDDEN) {
                        attemptRetryOnException("auth",
                                request, new AuthFailureError(networkResponse));
                    } else {
                        // TODO: Only throw ServerError for 5xx status codes.
                        throw new ServerError(networkResponse);
                    }
                } else {
                    throw new NetworkError(networkResponse);
                }
            }
        }
    }
```

这些代码里面重点看 httpResponse = mHttpStack.performRequest(request, headers); 这句是调用 mHttpStack 即最开始封装的HttpURLConnection或者HttpClient发送具体请求，往下部分是对httpResponse结果的处理。关于结果处理重点看 responseContents = entityToBytes(httpResponse.getEntity()); 这句通过entityToBytes方法将原entity转换成了byte[]。看一下具体的实现

``` java
/** Reads the contents of HttpEntity into a byte[]. */
    private byte[] entityToBytes(HttpEntity entity) throws IOException, ServerError {
        PoolingByteArrayOutputStream bytes =
                new PoolingByteArrayOutputStream(mPool, (int) entity.getContentLength());
        byte[] buffer = null;
        try {
            InputStream in = entity.getContent();
            if (in == null) {
                throw new ServerError();
            }
            buffer = mPool.getBuf(1024);
            int count;
            while ((count = in.read(buffer)) != -1) {
                bytes.write(buffer, 0, count);
            }
            return bytes.toByteArray();
        } finally {
            try {
                // Close the InputStream and release the resources by "consuming the content".
                entity.consumeContent();
            } catch (IOException e) {
                // This can happen if there was an exception above that left the entity in
                // an invalid state.
                VolleyLog.v("Error occured when calling consumingContent");
            }
            mPool.returnBuf(buffer);
            bytes.close();
        }
    }
```
可以重点看这句实现buffer = mPool.getBuf(1024); 这一句从 mPool 中取得一个 buffer ，然后将 entity 的内容写入这个 buffer 中。这个 mPool 是类 ByteArrayPool 的一个对象。ByteArrayPool 是一个字节数组缓存池。是用来缓存网络请求获得的数据。这个缓存池设置的目的就是方便缓存频繁的网络请求返回的结果，不然需要频繁的申请内存，释放内存，gc。频繁GC对客户端的性能有直接影响。ByteArrayPool 利用 mBuffersByLastUse 和 mBuffersBySize 完成字节数组的缓存，提供 getBuf 和 returnBuf的方法。当需要使内存区域的时候，先从已经分配的区域中获得以减少内存分配次数。当空间用完以后，再将数据返回给此缓冲区。
 **通过ByteArrayPool的使用，可以看到Volley所有的网络请求结果均是在缓存在内存中，可以很方便的频繁发送小的请求，但是对于一些大数据量的请求，会将这个缓存占满，所以volley不支持post大数据**


总结一下，volley有三类线程，MainThread、CacheDispatcher线程、NetworkDispatcher线程。volley 使用 RequestQueue 管理执行网络操作的工作线程。在主线程中调用 RequestQueue 的 add()方法来添加一条网络请求，这条请求会先被加入到缓存队列当中，如果发现可以找到相应的缓存结果就直接读取缓存并解析，然后回调给主线程。如果在缓存中没有找到结果，则将这条请求加入到网络请求队列中，然后处理发送HTTP请求，解析响应结果，写入缓存，并回调主线程。HTTP请求响应的结果会转换成byte[],使用ByteArrayPool这个对象来管理到缓存中，减少内存分配，释放次数。







### Retrofit


[Retrofit](https://github.com/square/retrofit)和okhttp一样是Square的开源项目。相比较volley，retrofit解耦的更加彻底。

- 使用注解来描述http请求
	- 请求方法注解
	- 请求头注解
	- 请求和响应格式注解
	- 请求参数注解
	
	以下是部分在接口HttpService中的代码，声明了一些接口
	
	```java
		//用户问题反馈接口
	    @FormUrlEncoded
	    @POST("/api/1.0/feedBack")
	    Observable<BaseHttpResult<JsonObject>> feedBack(@FieldMap Map<String, String> map);
	    
	    //上传图片接口
	    @Multipart
	    @POST("/api/1.0/upload")
	    Observable<BaseHttpResult<ImageUploadBean>> uploadImage(@PartMap Map<String, MultipartBody.Part> map);
	    
	    //获取消息列表
	    @GET("/api/1.0/messages")
	    Observable<BaseHttpResult<List<MessageBean>>> getMessageList(@Query("userID") String userId);
	
	```

- 可以配置不同的请求适配器，比如RxJava，Java8，Guava；可以配置不同的反序列化工具，比如json，protobuff，xml等；也可以配置不同的http请求客户端，默认使用okhttp。

	
	```java
	// http请求客户端
	client = new OkHttpClient
		.Builder()
		.addInterceptor(addQueryParameterInterceptor())  //参数添加
		.addInterceptor(addHeaderInterceptor()) // token过滤
		.addInterceptor(httpLoggingInterceptor) //日志,所有的请求响应度看到
		.cache(cache)  //添加缓存
		.connectTimeout(60l, TimeUnit.SECONDS)
		.readTimeout(60l, TimeUnit.SECONDS)
		.writeTimeout(60l, TimeUnit.SECONDS)
		.build();
	// 获取retrofit的实例
	retrofit = new Retrofit
		.Builder()
		.baseUrl(baseUrl) //配置baseurl
		.client(client) //配置http请求客户端
		.addCallAdapterFactory(RxJavaCallAdapterFactory.create()) //配置请求适配器，这里用Rxjava
		.addConverterFactory(GsonConverterFactory.create()) //配置反序列化工具，这里用Gson
		.build();
		
	```
	
在上面的示例代码中可以看到retrofit的网络请求部分依赖于okhttp，okhttp完成缓存，超时时间，日志拦截器配置等；请求的适配器，反序列化工具也是配置的其他开源工具。retrofit只负责一个架构组织的工作，将网络请求中用到的各个模块灵活的拼装起来，可以方便的更换各个组件。贯彻了单一职责的原则。

- 设计模式的使用

retrofit如此灵活的配置，主要得益于它的架构设计中使用了大量的设计模式。

![include](/assets/img/2017-10-24_adapt.png)
	
retrofit的源码可以说是设计模式的教科书，上面这张图中只列出了在retrofit中应用的部分设计模式，一些基本的如工厂模式，单例模式等，并未列出。

在retrofit中请求的适配器，反序列化工具的配置部分都用到了适配器模式。
	
其实适配器模式并不复杂,合理的使用会对代码的灵活性带来很大的收益。下图是适配器模式的类图。
	
![include](/assets/img/2017-10-24_adapter_mode.png)
	
我们来分析一下retrofit如何通过适配器模式来方便的切换不同的请求适配器。
	

retrofit的发送网络请求一般只需要两行代码

```java
	HttpService httpService = retrofit.create(GitHubService.class);//获得HttpService的一个实现类
	Observable<BaseHttpResult<List<MessageBean>>> messagetList = httpService.getMessageList("123") //调用上文的getMessageList方法
```

我们跟进retrofit的create方法


``` java
	public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            ServiceMethod serviceMethod = loadServiceMethod(method);
            OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }
	
```

在这个实现里，创建api HttpService 的实现实例使用到了[动态代理](http://a.codekk.com/detail/Android/Caij/%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%20Java%20%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86)。

我们重点分析最后面的三行代码。

第一行 ServiceMethod serviceMethod = loadServiceMethod(method); ServiceMethod 这个类的第一行注释很好的诠释了这个类的作用, 将一个 api 方法转换成为一个 http 请求。

```java
/** Adapts an invocation of an interface method into an HTTP call. */

final class ServiceMethod<T> {
  // Upper and lower characters, digits, underscores, and hyphens, starting with a character.
  static final String PARAM = "[a-zA-Z][a-zA-Z0-9_-]*";
  static final Pattern PARAM_URL_REGEX = Pattern.compile("\\{(" + PARAM + ")\\}");
  static final Pattern PARAM_NAME_REGEX = Pattern.compile(PARAM);

  final okhttp3.Call.Factory callFactory;
  final CallAdapter<?> callAdapter;

  private final HttpUrl baseUrl;
  private final Converter<ResponseBody, T> responseConverter;
  private final String httpMethod;
  private final String relativeUrl;
  private final Headers headers;
  private final MediaType contentType;
  private final boolean hasBody;
  private final boolean isFormEncoded;
  private final boolean isMultipart;
  private final ParameterHandler<?>[] parameterHandlers;

  ServiceMethod(Builder<T> builder) {
    this.callFactory = builder.retrofit.callFactory();
    this.callAdapter = builder.callAdapter;
    this.baseUrl = builder.retrofit.baseUrl();
    this.responseConverter = builder.responseConverter;
    this.httpMethod = builder.httpMethod;
    this.relativeUrl = builder.relativeUrl;
    this.headers = builder.headers;
    this.contentType = builder.contentType;
    this.hasBody = builder.hasBody;
    this.isFormEncoded = builder.isFormEncoded;
    this.isMultipart = builder.isMultipart;
    this.parameterHandlers = builder.parameterHandlers;
  }
  ...
```

在这里我们主要分析 ServiceMethod 中的两个成员 callAdapter 和 parameterHandlers 。callAdapter 是我们在 build retrofit 时候 addCallAdapterFactory(RxJavaCallAdapterFactory.create()) 提供的。parameterHandlers 由 ServiceMethod 中parseParameter 这个函数生成，负责对注解进行处理，构造为 http 请求。所以我们才可以在最开始方便的使用注解的方式来描述 http 请求。想了解注解如何处理的可以详细看一下parameterHandlers相关的函数

第二行 OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args) 
OkHttpCall执行具体的网络请求，并解析返回数据。OkHttpCall中进行网络请求的方法主要有两个： execute()执行同步请求，enqueue(Callback<T> callback) 执行异步请求。

第三行 return serviceMethod.callAdapter.adapt(okHttpCall) 回到了最开始说的适配器模式的使用，

这句话相当于上面类图中的client。

所有的 callAdapter 都实现接口 CallAdapter<T>，CallAdapter<T> 相当于类图中的Target

默认的 DefaultCallAdapterFactory 生成的 callAdapter 直接将 Call 返回，相当于类图中的 Adapter。


retfofit 中自带的 ExecutorCallAdapterFactory，这个工厂类生成的 CallAdapter 类将 Call<R> 适配成 ExecutorCallbackCall，ExecutorCallbackCall 可以支持回调。

``` java
	@Override public <R> Call<R> adapt(Call<R> call) {
		return new ExecutorCallbackCall<>(callbackExecutor, call);
	}
	
```
这是一个上述类图中Adaptee的实现

RxJavaCallAdapterFactory，工厂类生成的 SimpleCallAdapter 实现了CallAdapter，将 Call<R> 适配成 Observable，可以用 rxjava 进行相关操作，线程切换等，SimpleCallAdapter 也是上述类图中Adaptee的一种实现。

``` java
	@Override public <R> Observable<R> adapt(Call<R> call) {
	Observable<R> observable = Observable.create(new 	CallOnSubscribe<>(call))
		.lift(OperatorMapResponseToBodyOrError.<R>instance());
		if (scheduler != null) {
			return observable.subscribeOn(scheduler);
		}
		return observable;
	}
	
```
	
总结一下，retrofit 的特点是解耦的更加彻底。其解耦做的好的原因是因为它应用了大量的设计模式。我们分析了其 callAdatper 上面适配器模式的使用，我们可以实现或者包装自己的请求适配器，实现 CallAdaptet<T> 这个接口，并通过相应的 AdapterFactory 类集成到retrofit 中使用。通过这个分析应该对于 retrofit 网络请求的发送过程也有了进一步的了解。




# Reference
[Retrofit分析-漂亮的解耦套路](http://www.jianshu.com/p/45cb536be2f4)

[Android Volley完全解析(四)](http://blog.csdn.net/guolin_blog/article/details/17656437)

[Google Andorid developer doc](https://developer.android.com/develop/index.html)

















