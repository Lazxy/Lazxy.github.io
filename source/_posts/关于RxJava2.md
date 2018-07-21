---
title: 关于RxJava2
date: 2017-05-19 13:46:57
tags: 笔记
---

> 先前对RxJava的认识只停留在“它是可以与Retrofit配合的线程调度工具”的层面，并没有更多地探索它的原理和用法。然后在项目中要进行耗时文件I/O操作，正在考虑用AsynTask还是Handler来进行多线程处理回调时，突然想起了RxJava，遂决定再翻一遍以前看过的技术文章，认真地把它好好学一遍。（并且曾在项目里用过的RxJava由于依赖问题用的基本是RxJava1.x，不过既然RxJava2.x也已经发布稳定版，当然是学最新的啦。）

#### 配置依赖

```
compile 'io.reactivex.rxjava2:rxjava:2.1.0' 
compile 'io.reactivex.rxjava2:rxandroid:2.0.1'//RxAndroid更新可能会慢一点，但反正不要求版本号一致
//这里的包名也和1.x时代的 'io.reactivex:rxjava'有所区别了
```

<!--more-->

#### 基本用法

RxJava的事件与线程调度主要是通过观察者模式，而它最主要的两个对象自然也就是观察者和被观察者（或者叫被订阅者），及Observer和Observable，其承担的任务分别为事件处理与事件产生（或者说分发），其基本使用如下：

```
//这里的ObservableOnSubscribe与RxJava1.x中的Observable.OnSubscribe一样都是事件的分发者，不过1.x中的调用方法是call，参数为Subscriber
Observable.create(new ObservableOnSubscribe<T>(){
  @Override
  public void subscribe(ObservableEmitter<T> emitter) throws Exception{
    emitter.onNext(t1);
    emitter.onNext(t2);
    emitter.onComplete();//表示订阅关系结束，接下来的onNext操作将不会得到响应。该方法可以调用多次，但只有第一次起效
    //emitter.onError(); //表示异常出现，也会中断消息队列的发送，该方法只能调用一次，多次调用或与onComplete()共同出现会抛出异常
  }
}).subscribeOn(Schedulers.io()) //表示在I/O线程被订阅，或者说分发消息
  .observeOn(AndroidSchedulers.mainThread())//表示在主线程观察，或者说处理消息
  .onSubscribe(new Observer<T>() {
            @Override
            public void onSubscribe(Disposable d) {
				//作为第一个被调用的方法，可以保存其持有的Disposable对象，用于切断Observable与Observer间的关系，类似上面的onComplete()
            }

            @Override
            public void onNext(T value) {
				//响应上面的onNext()
            }

            @Override
            public void onError(Throwable e) {
				//响应上面的onError()
            }

            @Override
            public void onComplete() {
				//响应上面的onComplete()
            }
        });
```

上面表示了一个未进行额外操作的订阅模式操作，总的来说就是`确定消息分发逻辑` ->`确定线程关系`->`消息分发`->`消息处理`的一个过程。不过这是相对于一个消息队列的处理逻辑，如果仅仅是简单的想要执行一些异步任务的话，Observable与Observer都可以简化构造过程，如：

```
//传入一个或多个同一类型参数，最多10个，默认分发策略是依次onNext,最后onComplete
Observable.just(index) 
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(observer);
```

或者：

```
//直接传入一个数组,也是依次调用onNext，最后onComplete
Observable.from("I","Love","You")
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(observer);
```

Observer处理流程还可以直接写在链里：

```
...//同上
.doOnNext(new Consumer<String>() {
                    @Override
                    public void accept(@NonNull String s) throws Exception {
                        ...
                    }
                })
.doOnError(new Consumer<Throwable>() {
                    @Override
                    public void accept(@NonNull Throwable throwable) throws Exception {
                        
                    }
                })                   
.doOnComplete(new Action() {
                    @Override
                    public void run() throws Exception {
                    
                    }
                })
.doOnSubscribe(new Consumer<Disposable>() {
                    @Override
                    public void accept(@NonNull Disposable disposable) throws Exception {
                        
                    }
                })
.onSubscribe();//这里面也可以再写Observer对象，这样上面定义的方法和这里的Observer对象方法都会被调用一次，也可以传入对应的几个Consumer(或Action)，参数顺序同上。
```

这样以来，就可以省去一大部分不需要实现的流程与逻辑方法了，不过要进行异步操作，subscribeOn和observeOn两个方法当然是需要注意的，下面是需要了解的基础信息：

```
1.RxJava为这两个方法提供了几个预设值，分别为：
  Schedulers.io() 代表io操作的线程, 通常用于网络,读写文件等io密集型的操作
  Schedulers.computation() 代表CPU计算密集型的操作, 例如需要大量计算的操作
  Schedulers.newThread() 代表一个常规的新线程
  AndroidSchedulers.mainThread() 代表Android的主线程
  
2.subscribeOn被多次调用时，只有第一次起效，但observeOn被多次调用时，新的线程会起效。
PS：RxJava分发机制的执行顺序是与其构成链的顺序完全一致的，故在不同分发方法之间插入observeOn可以达到分离回调所处线程的效果。
```

到这里，RxJava的基本使用就差不多了。

#### 操作符

操作符是RxJava中相当重要的一部分，通过各种各样的操作符，能够轻易地完成各种对象的转换以及不同被观察者间的对接，可以将其看作适配器模式的一种使用。下面是一些常用的操作符(上面create、just等方法其实也是操作符的一种)：

**转换操作符**

```

1.Map
RxJava中最简单的操作符，其作用是将对象类型进行转换（这个操作其实在Observer端也可以直接实现，不过将其分离开来可以显得代码条例更清晰）。

示例：
//Integer->String的示例，此时Observable对象的泛型为Integer，但Observer的泛型为String
.map(new Function<Integer, String>() { 
      @Override
      public String apply(Integer integer) throws Exception {
      return "This is result " + integer;
   }

2.flatMap
用于两个Observable的嵌套操作，或者说将一种类型的Observable转化为另一种类型的Observable再继续分发，但这种转换分发的过程是无序的，可能会打乱队列原本的顺序。
示例：
//将一个Observable<Integer>转化为一个Observable<String>对象，并按原策略接下去分发。
.flatMap(new Function<Integer, ObservableSource<String>>() {
            @Override
            public ObservableSource<String> apply(Integer integer) throws Exception {
                final List<String> list = new ArrayList<>();
                for (int i = 0; i < 3; i++) {
                    list.add("I am value " + integer);
                }
                return Observable.fromIterable(list);
            }
            
3.concatMap
与flatMap基本相同，区别在于其是有序转换的。

4.buffer
可用于Observable值的缓存，可设定相应的缓存数量，缓存时间，以及缓存满后一次传送几个信息。
示例：
Observable.from(strings)
.buffer(strings.length)//全部缓存到数组并一次性输出到回调
//.buffer(strings.length,1)//全部缓存，一次输出一个
//.buffer(3, TimeUnit.SECONDS)//每隔3秒取出一次
.subscribe(new Observer<List<String>>...)      
```

**合并操作符**

```
1.zip:
按消息分发顺序一一对应地将两个Observable队列合并，其发送结果长度以两个队列中较短的那个为准，多余的消息并不会触发Observer的回调
示例：
//合并两个Observable队列，前者为Integer类型，后者为Sring类型，合成结果为String类型
Observable.zip(observable1, observable2, new BiFunction<Integer, String, String>() {           
@Override                                                                                    
public String apply(Integer integer, String s) throws Exception {                            
return integer + s;                                                                      
} 
//需要注意的是，这两个队列需要处于不同的线程才会有交替发送的现象，否则会等待其中一个队列把消息发完才开始发第二个队列的消息并合并

2.merge
按照消息分发顺序合并两个相同类型的队列，依次输出，不需要一一对应，且不进行任何中间操作。
示例：
Observable.merge(observable1,observable2);
```

**过滤操作符**

```
1.filter
设置过滤条件，禁止不符合条件的消息发送。
示例：
.filter(new Pridecate(){
  @Override
  boolean test(String str) throws Exception{
    return str.length()>5;//过滤长度小于5的字符串
  }
});

2.take
只选取前n个消息发送。
示例：
.take(100)//选取队列中前100个消息

3.skip
跳过前n个消息发送。
示例：
.skip(5)//跳过队列中的前5个消息
```




这些操作符的基础都是Observable类下的lift()，其实质是创建一个新的Observable来代理转换前的Observable，以此实现相应操作功能，这个机制也同样适用于Schedulers，可以参考解析：[给Android开发者的RxJava详解](http://gank.io/post/560e15be2dca930e00da1083).更多操作符，可以参见RxJava的Github页面[Oprators](http://reactivex.io/documentation/operators.html).

