# Retrofit解析

## 基本使用

首先要根据后台服务定义一个访问接口：

```java
public interface ServiceApi {
    @GET("j/search_subjects")
    Single<DoubanRes> request(@Query("type") String type, @Query("tag") String tag,
                              @Query("page_limit") int pageLimit, 
                              @Query("page_start") int pageStart);
}
```

然后创建retrofit实例，且构造出ServiceApi实例：

```java
String url = "https://movie.douban.com";
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl(url)
    .addConverterFactory(GsonConverterFactory.create())
    .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
    .build();
ServiceApi api = retrofit.create(ServiceApi.class);
```

最后，直接调用ServiceApi的request方法执行网路请求：

```java
api.request("tv", "热门", 50, 0)
    .subscribe(new SingleObserver<DoubanRes>() {
        @Override
        public void onSubscribe(Disposable d) {

        }

        @Override
        public void onSuccess(DoubanRes doubanRes) {
            System.out.println("onSuccess");
            System.out.println(new Gson().toJson(doubanRes));
        }

        @Override
        public void onError(Throwable e) {

        }
    });
```

请求结果如下：

```
onSuccess
{"subjects":[{"rate":"9.0","cover_x":2000,"title":"隐秘的角落","url":"https://movie.douban.com/subject/33404425/","playable":true,"cover":"https://img1.doubanio.com/view/photo/s_ratio_poster/public/p2609064048.jpg","id":"33404425","cover_y":2998,"is_new":false}...]}
```

## Retrofit实例化

我们先来看看Retrofit实例是如何构建的，在前面的例子中，我们通过Retrofit.Builder来创建：

```java
String url = "https://movie.douban.com";
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl(url)
    .addConverterFactory(GsonConverterFactory.create())
    .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
    .build();
```

可以看出，这里通过Builder模式构建Retrofit对象，在Builder对象中通过client(OkHttpClient client)、baseUrl(String baseUrl)、addConverterFactory(Converter.Factory factory)、addCallAdapterFactory(CallAdapter.Factory factory) 等方法设置参数，下面看看Builder的构造器和这些方法实现：

```java
Builder(Platform platform) {
    this.platform = platform;
    // Add the built-in converter factory first. This prevents overriding its behavior but also
    // ensures correct behavior when using converters that consume all types.
    converterFactories.add(new BuiltInConverters());
}

public Builder() {
    this(Platform.get());
}

public Builder client(OkHttpClient client) {
    return callFactory(checkNotNull(client, "client == null"));
}

public Builder callFactory(okhttp3.Call.Factory factory) {
    this.callFactory = checkNotNull(factory, "factory == null");
    return this;
}

public Builder baseUrl(String baseUrl) {
    checkNotNull(baseUrl, "baseUrl == null");
    HttpUrl httpUrl = HttpUrl.parse(baseUrl);
    if (httpUrl == null) {
        throw new IllegalArgumentException("Illegal URL: " + baseUrl);
    }
    return baseUrl(httpUrl);
}

public Builder baseUrl(HttpUrl baseUrl) {
    checkNotNull(baseUrl, "baseUrl == null");
    List<String> pathSegments = baseUrl.pathSegments();
    if (!"".equals(pathSegments.get(pathSegments.size() - 1))) {
        throw new IllegalArgumentException("baseUrl must end in /: " + baseUrl);
    }
    this.baseUrl = baseUrl;
    return this;
}

public Builder addConverterFactory(Converter.Factory factory) {
    converterFactories.add(checkNotNull(factory, "factory == null"));
    return this;
}

public Builder addCallAdapterFactory(CallAdapter.Factory factory) {
    adapterFactories.add(checkNotNull(factory, "factory == null"));
    return this;
}

public Builder callbackExecutor(Executor executor) {
    this.callbackExecutor = checkNotNull(executor, "executor == null");
    return this;
}

public Retrofit build() {
    if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
    }

    okhttp3.Call.Factory callFactory = this.callFactory;
    if (callFactory == null) {
        callFactory = new OkHttpClient();
    }

    Executor callbackExecutor = this.callbackExecutor;
    if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
    }

    // Make a defensive copy of the adapters and add the default Call adapter.
    List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
    adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

    // Make a defensive copy of the converters.
    List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);

    return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
                        callbackExecutor, validateEagerly);
}
```

Retrofit.Builder在新建实例的时候会记录当前运行的平台，不同平台的默认线程等有所不同，比如android平台上默认运行线程为UI线程。然后在各个参数方法中主要就是将用户设置的参数保存下来，在调用build()方法之后使用这些设置的参数构造Retrofit对象。

## Service接口构造

在构造完Retrofit对象之后，就需要构造Service接口对象了，起始这里是是使用动态代理来构造自定义的Service接口实例的，如下：

```
public <T> T create(final Class<T> service) {
  Utils.validateServiceInterface(service);
  if (validateEagerly) {
    eagerlyValidateMethods(service);
  }
  return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
      new InvocationHandler() {
        private final Platform platform = Platform.get();

        @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
            throws Throwable {
          // If the method is a method from Object then defer to normal invocation.
          if (method.getDeclaringClass() == Object.class) {
            return method.invoke(this, args);
          }
          if (platform.isDefaultMethod(method)) {
            return platform.invokeDefaultMethod(method, service, proxy, args);
          }
          ServiceMethod<Object, Object> serviceMethod =
              (ServiceMethod<Object, Object>) loadServiceMethod(method);
          OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
          return serviceMethod.callAdapter.adapt(okHttpCall);
        }
      });
}
```

这里参数也就是interface的class对象，从扇面代码不难看出，Service接口实例是通过java动态代理，构造了一个代理对象，然后将代理对象强制转换成对应的接口类型。

## 接口实现

在构造完Service接口的实例之后，我们就可以直接通过接口方法发起请求了：

```java
api.request("tv", "热门", 50, 0)
```

前面说到了Service接口实际上是一个动态代理的对象，那么当调用此代理对象上的方法时，必定会调用代理对象的invoke()方法，因此具体的实现就在于代理对象的invoke方法中：

```java
public <T> T create(final Class<T> service) {
  Utils.validateServiceInterface(service);
  if (validateEagerly) {
    eagerlyValidateMethods(service);
  }
  return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
      new InvocationHandler() {
        private final Platform platform = Platform.get();

        @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
            throws Throwable {
          // If the method is a method from Object then defer to normal invocation.
          if (method.getDeclaringClass() == Object.class) {
            return method.invoke(this, args);
          }
          if (platform.isDefaultMethod(method)) {
            return platform.invokeDefaultMethod(method, service, proxy, args);
          }
          ServiceMethod<Object, Object> serviceMethod =
              (ServiceMethod<Object, Object>) loadServiceMethod(method);
          OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
          return serviceMethod.callAdapter.adapt(okHttpCall);
        }
      });
}
```

很容易看出，在调用代理方法时，会跳过invoke里面的连个if语句，直接执行下面的方法：

```java
@Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
    throws Throwable {
    ......
    ServiceMethod<Object, Object> serviceMethod =
        (ServiceMethod<Object, Object>) loadServiceMethod(method);
    OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
    return serviceMethod.callAdapter.adapt(okHttpCall);
}
```

loadServiceMethod(method)是用来查找当前调用方法所对应的的ServiceMethod对象，以便后面发送网络请求，那我们就看看它的实现：

```java
private final Map<Method, ServiceMethod<?, ?>> serviceMethodCache = new ConcurrentHashMap<>();

ServiceMethod<?, ?> loadServiceMethod(Method method) {
    ServiceMethod<?, ?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    synchronized (serviceMethodCache) {
        result = serviceMethodCache.get(method);
        if (result == null) {
            result = new ServiceMethod.Builder<>(this, method).build();
            serviceMethodCache.put(method, result);
        }
    }
    return result;
}
```

loadServiceMethod中先从缓存的serviceMethod中查找当前调用的方法，如果缓存中不存在当前方法，则会通过

ServiceMethod.Builder来构造一个ServiceMethod，同时将其加入缓存，后续再调用相同方法时就可以直接使用缓存好的对象了。

下面来看下ServiceMethod如何构造：

```java
Builder(Retrofit retrofit, Method method) {
    this.retrofit = retrofit;
    this.method = method;
    this.methodAnnotations = method.getAnnotations();
    this.parameterTypes = method.getGenericParameterTypes();
    this.parameterAnnotationsArray = method.getParameterAnnotations();
}

public ServiceMethod build() {
    callAdapter = createCallAdapter();
    responseType = callAdapter.responseType();
    if (responseType == Response.class || responseType == okhttp3.Response.class) {
        throw methodError("'"
                          + Utils.getRawType(responseType).getName()
                          + "' is not a valid response body type. Did you mean ResponseBody?");
    }
    responseConverter = createResponseConverter();

    for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
    }

    if (httpMethod == null) {
        throw methodError("HTTP method annotation is required (e.g., @GET, @POST, etc.).");
    }

    if (!hasBody) {
        if (isMultipart) {
            throw methodError(
                "Multipart can only be specified on HTTP methods with request body (e.g., @POST).");
        }
        if (isFormEncoded) {
            throw methodError("FormUrlEncoded can only be specified on HTTP methods with "
                              + "request body (e.g., @POST).");
        }
    }

    int parameterCount = parameterAnnotationsArray.length;
    parameterHandlers = new ParameterHandler<?>[parameterCount];
    for (int p = 0; p < parameterCount; p++) {
        Type parameterType = parameterTypes[p];
        if (Utils.hasUnresolvableType(parameterType)) {
            throw parameterError(p, "Parameter type must not include a type variable or wildcard: %s",
                                 parameterType);
        }

        Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
        if (parameterAnnotations == null) {
            throw parameterError(p, "No Retrofit annotation found.");
        }

        parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
    }

    if (relativeUrl == null && !gotUrl) {
        throw methodError("Missing either @%s URL or @Url parameter.", httpMethod);
    }
    if (!isFormEncoded && !isMultipart && !hasBody && gotBody) {
        throw methodError("Non-body HTTP method cannot contain @Body.");
    }
    if (isFormEncoded && !gotField) {
        throw methodError("Form-encoded method must contain at least one @Field.");
    }
    if (isMultipart && !gotPart) {
        throw methodError("Multipart method must contain at least one @Part.");
    }

    return new ServiceMethod<>(this);
}
```

build()方法中，主要有以下流程

1. 调用createCallAdapter()创建CallAdapter对象，内部就是根据在Retrofit.Builder中调用addCallAdapterFactory()添加的工厂进行构造的
2. 调用createResponseConverter()创建Converter对象，这也是通过addConverterFactory()中添加的的工厂创建的
3. 遍历当前方法的注解，调用parseMethodAnnotation(annotation)解析每个注解，并将注解中的url等参数保存在当前对象中。
4. 遍历当前方法的参数，调用parseParameter(p, parameterType, parameterAnnotations)解析参数中的注解，并添加对应的parameterHandler，以便在发送请求时将对应的参数转换为HTTP请求参数。
5. 使用当前Builder对象构建ServiceMethod对象，将其返回。

其中每个步骤的详细实现就不作细致的分析了。

再构建完ServiceMethod之后，就需要开始分析HTTP请求的发送流程了，这里再回到Retrofit的create方法中：

```java
@Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
    throws Throwable {
    ......
    ServiceMethod<Object, Object> serviceMethod =
        (ServiceMethod<Object, Object>) loadServiceMethod(method);
    OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
    return serviceMethod.callAdapter.adapt(okHttpCall);
}
```

这里先创建了一个OKHttpCall实例，然后将其传给serviceMethod.callAdapter.adapt()方法。我们先看看OkHttpCall有哪些方法：

```java
final class OkHttpCall<T> implements Call<T> {
  @Override public synchronized Request request() {
    ......
  }

  @Override public void enqueue(final Callback<T> callback) {
    ......
  }

  @Override public synchronized boolean isExecuted() {
    return executed;
  }

  @Override public Response<T> execute() throws IOException {
    ......
  }

  public void cancel() {
   ......
  }

  @Override public boolean isCanceled() {
    ......
  }
}
```

OkHttpCall封装了okhttp库，是真正发送网络请求的类，包含enqueue和execute方法，其中enqueue表示发起异步请求，而execute表示发起同步请求。我们暂时先不看它的具体实现，先往下看。

```java
serviceMethod.callAdapter.adapt(okHttpCall);
```

这里使用使用适配器模式将OKHTTP网络请求结果转换为在Service接口中指定的返回值类型。

callAdapter就是在Retrofit.Builder中addCallAdapterFactory()中添加的factory构建的，这里我们添加的是RxJava2CallAdapterFactory工厂，所以这里实际上构建的是RxJava2CallAdapter对象，这里我们来看看它的adapt()实现：

```java
@Override public Object adapt(Call<R> call) {
    Observable<Response<R>> responseObservable = isAsync
        ? new CallEnqueueObservable<>(call)
        : new CallExecuteObservable<>(call);

    Observable<?> observable;
    if (isResult) {
        observable = new ResultObservable<>(responseObservable);
    } else if (isBody) {
        observable = new BodyObservable<>(responseObservable);
    } else {
        observable = responseObservable;
    }

    if (scheduler != null) {
        observable = observable.subscribeOn(scheduler);
    }

    if (isFlowable) {
        return observable.toFlowable(BackpressureStrategy.LATEST);
    }
    if (isSingle) {
        return observable.singleOrError();
    }
    if (isMaybe) {
        return observable.singleElement();
    }
    if (isCompletable) {
        return observable.ignoreElements();
    }
    return observable;
}
```

adapt中根据call对象和是否异步来构造CallEnqueueObservable或者CallExecuteObservable，然后将其封装成合适的Observable对象，作为结果返回。这里重点在CallEnqueueObservable，我们来看看这两个类的实现：

```java
final class CallEnqueueObservable<T> extends Observable<Response<T>> {
  private final Call<T> originalCall;

  CallEnqueueObservable(Call<T> originalCall) {
    this.originalCall = originalCall;
  }

  @Override protected void subscribeActual(Observer<? super Response<T>> observer) {
    // Since Call is a one-shot type, clone it for each new observer.
    Call<T> call = originalCall.clone();
    CallCallback<T> callback = new CallCallback<>(call, observer);
    observer.onSubscribe(callback);
    call.enqueue(callback);
  }

  private static final class CallCallback<T> implements Disposable, Callback<T> {
    private final Call<?> call;
    private final Observer<? super Response<T>> observer;
    boolean terminated = false;

    CallCallback(Call<?> call, Observer<? super Response<T>> observer) {
      this.call = call;
      this.observer = observer;
    }

    @Override public void onResponse(Call<T> call, Response<T> response) {
      if (call.isCanceled()) return;

      try {
        observer.onNext(response);

        if (!call.isCanceled()) {
          terminated = true;
          observer.onComplete();
        }
      } catch (Throwable t) {
        if (terminated) {
          RxJavaPlugins.onError(t);
        } else if (!call.isCanceled()) {
          try {
            observer.onError(t);
          } catch (Throwable inner) {
            Exceptions.throwIfFatal(inner);
            RxJavaPlugins.onError(new CompositeException(t, inner));
          }
        }
      }
    }

    @Override public void onFailure(Call<T> call, Throwable t) {
      if (call.isCanceled()) return;

      try {
        observer.onError(t);
      } catch (Throwable inner) {
        Exceptions.throwIfFatal(inner);
        RxJavaPlugins.onError(new CompositeException(t, inner));
      }
    }

    @Override public void dispose() {
      call.cancel();
    }

    @Override public boolean isDisposed() {
      return call.isCanceled();
    }
  }
}
```

subscribeActual中调用了call.enqueue()方法发起请求，并将CallCallback作为毁掉参数传递进去了，而CallCallback的实现就是将请求结果通过observer发送给用户。

下面我们看一下CallExecuteObservable的实现：

```java
final class CallExecuteObservable<T> extends Observable<Response<T>> {
  private final Call<T> originalCall;

  CallExecuteObservable(Call<T> originalCall) {
    this.originalCall = originalCall;
  }

  @Override protected void subscribeActual(Observer<? super Response<T>> observer) {
    // Since Call is a one-shot type, clone it for each new observer.
    Call<T> call = originalCall.clone();
    observer.onSubscribe(new CallDisposable(call));

    boolean terminated = false;
    try {
      Response<T> response = call.execute();
      if (!call.isCanceled()) {
        observer.onNext(response);
      }
      if (!call.isCanceled()) {
        terminated = true;
        observer.onComplete();
      }
    } catch (Throwable t) {
      Exceptions.throwIfFatal(t);
      if (terminated) {
        RxJavaPlugins.onError(t);
      } else if (!call.isCanceled()) {
        try {
          observer.onError(t);
        } catch (Throwable inner) {
          Exceptions.throwIfFatal(inner);
          RxJavaPlugins.onError(new CompositeException(t, inner));
        }
      }
    }
  }
}
```

可以看出，这里就是调用call.execute()，并将请求结果通过observer发送给用户。

到目前为止，retrofit的实现基本已经很清楚了，不过前面提到的OkHttpCall的实现还没有细看，这里就来分析一下：

```java
final class OkHttpCall<T> implements Call<T> {
  ......
      
  @Override public void enqueue(final Callback<T> callback) {
    checkNotNull(callback, "callback == null");

    okhttp3.Call call;
    Throwable failure;

    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      call = rawCall;
      failure = creationFailure;
      if (call == null && failure == null) {
        try {
          call = rawCall = createRawCall();
        } catch (Throwable t) {
          failure = creationFailure = t;
        }
      }
    }

    if (failure != null) {
      callback.onFailure(this, failure);
      return;
    }

    if (canceled) {
      call.cancel();
    }

    call.enqueue(new okhttp3.Callback() {
      @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse)
          throws IOException {
        Response<T> response;
        try {
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          callFailure(e);
          return;
        }
        callSuccess(response);
      }

      @Override public void onFailure(okhttp3.Call call, IOException e) {
        try {
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }

      private void callFailure(Throwable e) {
        try {
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }

      private void callSuccess(Response<T> response) {
        try {
          callback.onResponse(OkHttpCall.this, response);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }
    });
  }

  @Override public Response<T> execute() throws IOException {
    okhttp3.Call call;

    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      if (creationFailure != null) {
        if (creationFailure instanceof IOException) {
          throw (IOException) creationFailure;
        } else {
          throw (RuntimeException) creationFailure;
        }
      }

      call = rawCall;
      if (call == null) {
        try {
          call = rawCall = createRawCall();
        } catch (IOException | RuntimeException e) {
          creationFailure = e;
          throw e;
        }
      }
    }

    if (canceled) {
      call.cancel();
    }

    return parseResponse(call.execute());
  }
    
  private okhttp3.Call createRawCall() throws IOException {
    Request request = serviceMethod.toRequest(args);
    okhttp3.Call call = serviceMethod.callFactory.newCall(request);
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
  }

  ......

  Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();

    // Remove the body's source (the only stateful object) so we can pass the response along.
    rawResponse = rawResponse.newBuilder()
        .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
        .build();

    int code = rawResponse.code();
    if (code < 200 || code >= 300) {
      try {
        // Buffer the entire body to avoid future I/O.
        ResponseBody bufferedBody = Utils.buffer(rawBody);
        return Response.error(bufferedBody, rawResponse);
      } finally {
        rawBody.close();
      }
    }

    if (code == 204 || code == 205) {
      rawBody.close();
      return Response.success(null, rawResponse);
    }

    ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
    try {
      T body = serviceMethod.toResponse(catchingBody);
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      // If the underlying source threw an exception, propagate that rather than indicating it was
      // a runtime exception.
      catchingBody.throwIfCaught();
      throw e;
    }
  }

  ......
}
```

在enqueue和execute方法，他们都会将serviceMethod转换成OKHTTP的Request对象，这是通过serviceMethod.toRequest(args)方法实现的:

```java
  Request toRequest(@Nullable Object... args) throws IOException {
    RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl, headers,
        contentType, hasBody, isFormEncoded, isMultipart);

    @SuppressWarnings("unchecked") // It is an error to invoke a method with the wrong arg types.
    ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;

    int argumentCount = args != null ? args.length : 0;
    if (argumentCount != handlers.length) {
      throw new IllegalArgumentException("Argument count (" + argumentCount
          + ") doesn't match expected count (" + handlers.length + ")");
    }

    for (int p = 0; p < argumentCount; p++) {
      handlers[p].apply(requestBuilder, args[p]);
    }

    return requestBuilder.build();
  }
```

他是通过ParameterHandler进行这个转换的。

在转换成OKHTTP的request对象之后，通过OKHTTP请求接口进行网络请求。

在拿到请求结果之后，又都通过parseResponse()方法解析结果，其中最关键的就是通过`serviceMethod.toResponse(catchingBody);`将OKHTTP的ResponseBody对象转换成接口中指定的类型，其内部使用convector实现：

```java
/** Builds a method return value from an HTTP response body. */
R toResponse(ResponseBody body) throws IOException {
    return responseConverter.convert(body);
}
```

我们例子中使用的是GsonConverterFactory，其中的response的converter是GsonResponseBodyConverter，我们看一下其中的实现：

```java
final class GsonResponseBodyConverter<T> implements Converter<ResponseBody, T> {
  private final Gson gson;
  private final TypeAdapter<T> adapter;

  GsonResponseBodyConverter(Gson gson, TypeAdapter<T> adapter) {
    this.gson = gson;
    this.adapter = adapter;
  }

  @Override public T convert(ResponseBody value) throws IOException {
    JsonReader jsonReader = gson.newJsonReader(value.charStream());
    try {
      return adapter.read(jsonReader);
    } finally {
      value.close();
    }
  }
}
```

可以看出，这中间使用gson将response转换为指定的对象了。