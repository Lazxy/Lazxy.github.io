---
title: 关于Retrofit
date: 2017-05-01 19:25:52
tags: 笔记
---

> 笔记转移之路的第一篇，就先好好回顾下这默默工作在层层封装下的劳模。

**Retrofit**同**OkHttp**一样都是square公司（然后查了一下发现Picasso和Dragger也是他们的项目，向大佬低头）的开源网络框架，大概因为这个原因，它们也有相似的Builder链式调用和相似的请求流程。当然Retrofit实际可以算作是OkHttp的加强版，其不仅有各种方便的注解简化代码，更有其他开源库的支持（看看漫山遍野的Retrofit+RxJava使用教程），下面简单地记录一下当时学习过的一些点。

<!--more-->

#### 添加依赖

```
compile 'com.squareup.retrofit2:retrofit:2.2.0'
compile 'com.squareup.okhttp3:okhttp:3.7.0'
//compile 'com.squareup.retrofit2:converter-gson:2.2.0'
//compile 'com.squareup.retrofit2:adapter-rxjava:2.2.0'
//compile 'io.reactivex:rxjava:1.1.7' //这里如果也用rxjava 2.0以上版本会和上面的adapter-rxjaava出现不同版本库冲突问题。
//compile 'io.reactivex:rxandroid:1.2.1'
```



#### 基本用法

```
Retrofit retrofit = new Retrofit.Builder()
                .baseUrl("http://lazxy.github.io/") //注意这里一定要以“/”结尾
                .client(client)
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create())//和RxJava一起用
                .addConverterFactory(GsonConverterFactory.create())//和Gson一起用
                .build();
```

实际上从这个Builder链就能大致看出Retrofit和OkHttp的关系了，Rerofit需要通过OkHttpClient来创建对象，其实也就是等于在它之上做了一层封装，除了OkHttp本身的各种属性之外，又给附上了适配器和对象转换。然而光有Retrofit对象并没有什么用，通道是搭好了，但请求对象呢？在OkHttp里，需要另外创建一个Request对象，在里面注入Url、请求头和各种Http请求方法，然而在Retrofit里，这一步直接换成了高大上的注解实现：

```
interface DemoService {
@GET("demo/{id}")
Call<ResponseBody> test(@Path("id") int id);
}

...
Call<ResponseBody> call = retrofit.create(DemoService.class);
call.enqueue(new Callback(){...})
```

这里直接通过向retrofit的create(Class class)方法传入接口类字节码实现反射调用创建了Call对象，也就是请求的载体，再和OkHttp一样，用一个enqueue方法把请求加入队列，就完成了请求的调用过程。其中请求的Http请求类型通过注解@GET注入，类似的还有下列注解：

| 请求类型    | 说明                                       |
| ------- | ---------------------------------------- |
| GET     | 通过请求获得响应，最多携带2048字节信息                    |
| POST    | 发送数据给服务器端，数据长度无限制，但非幂等，反复请求会造成数据的反复创建。   |
| PUT     | 更新或创建服务器端的数据，与POST的区别在于PUT的地址是精确到单个资源的，而POST只能作用于资源集合层面。（即前者能够针对单一文件进行操作，但后者只能在目录下进行创建操作） |
| DELETE  | 删除服务器端数据。                                |
| HEAD    | 同GET相同，但响应体为空，多用于测试连接性和信息更新。             |
| OPTIONS | 获取服务器端支持的请求方法，或者用于检查服务器的性能               |
| PATCH   | 更新服务器端数据的局部，可以是一个或多个字段，与PUT不同，PATCH操作是非幂等的，因为局部更新需要服务器程序配合，可能会造成资源改变。 |
| HTTP    | 通用型注解，可以用来表示上面七种请求类型。                    |

上面几种请求就是基本的几种Http请求，注解内的值表示retrofit创建时赋予的主路径(baseUrl)下的子路径**（当注解值仅为路径名时实际路径为baseUrl+注解值；而当注解值以“/”开头时实际路径为baseUrl的域名+注解值，忽略baseUrl中已有的子路径）**，**但当注解值为全路径且与baseUrl指向的域不同时，以注解值为准。**中间动态参数用`{param}`标注，在注解方法的参数列表中需要出现一一对应的@Path("param")注解来注入具体值。另附@HTTP注解的例子：

```
@HTTP(method = "get", path = "demo/{id}", hasBody = false) （替代上面的get）
```

另外还有以下几个相关注解：

| 注解名            | 说明                                       |
| -------------- | ---------------------------------------- |
| FormUrlEncoded | 表示请求体是一个Form表单，实际作用方式为设置请求头的Content-Type:application/x-www-form-urlencoded |
| Multipart      | 表示请求体支持分段传输，常用于文件上传，实际为设置请求头Content-Type:multipart/form-data |
| Streaming      | 表示响应体数据以流的形式返回，而不进入内存，在返回大容量数据时会用到       |
| Headers        | 用于添加请求头，其注解参数即为要添加的请求头键值对，可一次添加多个头       |

好了，上面提到的注解都是用于注解函数的，而对于函数中的各个参数，则采用下列注解：

| 注解名      | 说明                                       |
| -------- | ---------------------------------------- |
| Header   | 用于注解需要添加的头，注解参数为键，被注解的函数参数为值。            |
| Body     | 用于非表单请求体，即自定义对象，在发起请求的过程中会被相应的Convert序列化为预设的格式（如Json），并且其结果直接被设置为Request的请求体。 |
| Field    | 用于注解单个表单字段，使用时需要有FormUrlEncoded注解或者相应头表示其请求体为表单格式。 |
| FieldMap | 用于注解多个表单字段的集合，只接受Map<String,String>类型，其他类型会被强制调用toString()方法，使用时需要有FormUrlEncoded注解或者相应头表示其请求体为表单格式。 |
| Part     | 用于注解单个分段表单字段，支持类型为RequestBody、MultiPartBody.Part(内部表示)与其他任意类型(也是通过Convert转换),使用时需要有Multipart注解或者相应头表示其请求体为分段表单格式。 |
| PartMap  | 用于注解多个分段表单字段，只接受Map<String,RequestBody>类型，非RequestBody类型会通过Converter转换，使用时需要有Multipart注解或者相应头表示其请求体为分段表单格式。 |
| Path     | 用于注解Url路径相关参数，其参数与上述Http请求方法中的`{param}`中参数一一对应。 |
| Query    | 注解Url的附带参数值，其注解参数为键，被注解函数为值，注解结果会显示在最终请求的的路径上，形如?name=Lazxy。其实这个结果与Field最终表现出来的结果是相同的，但区别在于后者作用对象是请求体而非Url。 |
| QueryMap | 同FieldMap,区别也同样在于作用点为URL                 |
| Url      | 用于注解Url路径，当该注解存在时，Http方法会直接以该注解值为Url     |



Path、Query、Field几类注解没什么好讲的，其内部处理其实就是简单的在请求体或者直接在URL中加入key=value形式的值，这里着重讲一下Part。

上面说到了Part用于表示分段表单字段，然而这个分段是怎么实现的呢？其实际请求内容如下：

```
...
Content-Type: multipart/form-data; boundary=${bound}  //请求头里的值，@Multipart注解的功劳，这里的boundary用于下面分段的分割标识
//一个空行，表示请求头结束了，下面是请求体内容
--${bound}
Content-Disposition: form-data; name="file_name" //规定格式及请求字段
Content-Type: text/plain; charset=UTF-8 //该段请求体的类型以及其编码格式
Content-Transfer-Encoding: 8bit //解码转化方式
//一个空行，表示前面的信息段结束，下面一行是文件内容
avatar.jpg //Filename的内容
--${bound}
Content-Disposition: form-data; name="avatar"; filename="avatar.png"//name为请求字段，filename为建议服务器端保存的文件名
Content-Type: image/png

```

而Part注解的注解参数正是`Content-Disposition：form-data;`后面的部分，例如当需要传输一个文件时，它的参数可以这样写

```
Call<ResponseBody> uploadFile(@Part("file\";filename=\"file.txt"\"") RequestBody file);
//这里的三个转义字符是为了将其原本的name="${param}"格式强行拓展为name="${param1}";filename="${param2}"
```

而RequestBody的构造则写在调用这个请求的地方：

```
RequestBody fileRequest = RequestBody.create(MediaType.parse("text/plain"),new File("0/storage/file.txt"));//这里规定了文件类型，也就是上面的Type-Content
retrofit.create(Demo.class).uploadFile(fileRequest);
```

而当需要上传多个文件或者附加字段时，用PartMap就行，其中key为name，value为包装着文件的RequestBody或者基本类型（包括String，毕竟Converter默认都有String转化的方法）。

**PS**:Path推荐只用在与路径有关时，包括数据的部分最好用Query解决，而数据量较大或者需要不可见时建议用Filed提交。另外Query、Field和Part都支持数组或者实现了Iterable等可迭代对象导入。



#### 关于CallAdapter和Converter

​	在最基本的使用中，我们定义的接口及其内被注解的方法返回值类型为Call<ResponeBody>，而在正常的使用中，我们更希望能够返回一个我们自定义的对象类型从而更容易地将其打包或解析为请求体，这时就需要用到Retrofit的Converter机制了。Retrofit能够通过addConverterFactory方法来添加相应的Converter工厂类，并在**需要解析**时（包括对@Body注解对象转换为ResponseBody和ResponseBody转换为相应对象）创建相应的实现了Converter<F,T>接口的类，来进行对象类型转换，从而得到Call<Object>(这样便没有办法获得响应头或者状态码)、Call<Result<Object>>(这里是retrofit2.adapter.rxjava.Result)或者Call<Response<Object>>(这里是retrofit2.response)对象。

​	Converter<F,T>实现类的结构也很简单，主要是重写一个convert方法完成由F对象到T对象的转换。而Converter.Factory接口中定义了两个responseBodyConverter，分别用于ResponseBody对目标类的转换以及目标类对ResponseBody的转换，返回值分别为Converter<ResponseBody,?>和Converter<?,ResponseBody>；以及一个stringConverter，用于@Field、@Path等注解参数到String的转换，返回值自然是一个Converter<?,String>（当上述三个方法返回null时则代表无法解析相应类型）。所以Converter的工作原理就很清楚了，**当需要类型转换时，Converter.Factory会调用相应的工厂方法，创建对应转换方向的Converter对象，Converter对象再调用自身的convert方法完成实际转换工作。**

> 需要注意的是Converter的添加是有优先级的，当多个Converter都能对同一对象进行转换时，会优先调用先添加的Converter。而GsonConveter是不区分对象类型的，故当需要特别地解析某个对象的，需要将其Converter先于GsonConverter添加，否则会报类型不匹配的错误。



​	事实上，除了Call<T>中的T可以被代换，Call本身也是可以被替换的，这也是Retrofit与RxJava这一对能够处处展现其相性的前提。对于响应载体的替换需要用到CallAdapter，其实现与Converter相似，也是需要实现CallAdapter<T>，并且实现CallAdapter.Factory工厂类，最后用addAdapterFactory方法添加，这样一来自定义接口中就可以返回自定义响应载体对象了。

​	CallAdapter<T>的接口又定义了些什么呢？它首先有两个方法，responseType和adapt，这两个方法很好理解，前者用于返回响应体类型，即上面Converter定义的对象类型；而后者用于返回CallAdpater用于代理Call的类型，即其泛型T，它接受一个Call参数，相当于上面的ResponseBody，是原始响应载体。当然前面已经提到CallAdapter同样有一个内部工厂类，其中有get、getParameterUpperBound和getRawType三个方法。get自然是用于取得一个对象，其返回值便是需要返回的响应载体类型，如果无法进行合适的转换则返回null；getParameterUpperBound方法用于获取响应体类型，与responseType的返回值相同；getRawType返回的是T的Class对象。CallAdapter类的创建过程也大致可以归结为：**生成响应载体时，调用CallAdapter.Factory的get方法，传入需要的类型参数，如Observable<Entity>，首先通过getRawType确认是否为Observable，再通过getParameterUpperBound获取其泛型类型Entity，在构造CallAdapter实现类时传入，从而使responseType方法能获得正确的泛型类型，最后通过adapt返回目标响应载体对象。**



#### 最后

本来是想Retrofit和Gson一起列在这篇里的，不过貌似放在一起并不是很合适，Gson就下一次再搬喽，大概还有~~封装好了之后就再也没好好看过的~~ RxJava？ 