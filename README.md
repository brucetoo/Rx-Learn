# Rx-Learn
Rxjava - Rxandroid基础到高级学习

内容大部分来自 [Dan Lew](http://blog.danlew.net/2014/09/15/grokking-rxjava-part-1/) 的博客
               [抛物线](http://gank.io/post/560e15be2dca930e00da1083#toc_1)的博客
以及自己的一些理解

# Rxjava基本的概念
  `Rxjava` 中重要的两个类 `Observable` 和 `Subscribers`,及被观察者和观察者(订阅者)
  
  > Rx和标准 Observer pattern的一个重要的区别在于 Rx中 Observable(被观察者)不会发射任何items,直到有 Subscriber(订阅者及观察者)订阅
  
## Observable
 
  及`被观察者`或者说是 `消费者` 负责发射数据源,下面是简单的创建一个 Observable  
 
 ```java
 
     Observable<String> myObservable = Observable.create(
             new Observable.OnSubscribe<String>() {
                 @Override
                 public void call(Subscriber<? super String> sub) {
                 //发射string类型数据
                     sub.onNext("I am a Observable");
                 //数据发射完后 发射complete    
                     sub.onCompleted();
                 }
             }
         );
  
 ```
    
## Subscriber
   
   及`观察者`或者说是`订阅者` 负责消费`Observable`发射的数据源 
   `Subscriber`是`Observer`(观察者)的扩展，多了 `onStart` 回调 和 `unSubscribe` 取消订阅的方法
   `onStart` 是在 被观察者(`Observable`)事件还没被发送之前调用(`onNext`前),适合用于初始化
   某些操作的时候调用,但是**不适合做耗时的操作**(`onStart`执行的线程是和调用线程为同一个线程)
   如果想在同一时机执行异步的操作 可以用 `doOnSubscribe` 并且在后续跟上 `subscribeOn` 来指定线程
   
   `unSubscribe` 是 `Observer`继承了 `Subscription` 接口 提供了可以取消订阅的方法,避免内存泄露
   
   ```java
    
    Subscriber<String> mySubscriber = new Subscriber<String>() {
        @Override
        public void onNext(String s) { 
        System.out.println(s); 
        }
    
        @Override
        public void onCompleted() { }
    
        @Override
        public void onError(Throwable e) { }
    };
    
    // Subscriber分完整的回调和不完整的回调,比如说 可以只关心onNext,onCompleted,onError中的单个或者多个
   
   ```
   
## Subscription  
 
   及`Subscriber` 和 `Observable`发生  `subscribe`关系后的返回值
   主要用于对数据流进行随时的 `unSubscribe`处理
   
   ```java
    
       myObservable.subscribe(mySubscriber);
   
   ```
   
## Subject
   
   `Subject` 既可以是一个 `Observable` 也可以是一个 `Subscriber` :作为连接两者之间的桥梁
   一个 `Subject` 可以`订阅` 一个`Observable(被观察者)`,此时等同于 `Subscriber(观察者/订阅者)`,
   可以发射数据,同时也可以扮演 `Observable` 来 `消费` 自己发射的数据
   
   经典的一个用法就是用 `Subject`来实现 `EventBus` [Subject实现EventBus](http://nerds.weddingpartyapp.com/tech/2014/12/24/implementing-an-event-bus-with-rxjava-rxbus/);
   
   ```java
   
      // this is the middleman object
      public class RxBus {
      
        private final Subject<Object, Object> _bus = new SerializedSubject<>(PublishSubject.create());
      
        public void send(Object o) {
          _bus.onNext(o);
        }
      
        public Observable<Object> toObserverable() {
          return _bus;
        }
      }
      
      //Send event somewhere
       _rxBus.send(new TapEvent());
       
       
      //Litsten to event
      // note that it is important to subscribe to the exact same _rxBus instance that was used to post the events
       _rxBus.toObserverable()
           .subscribe(new Action1<Object>() {
             @Override
             public void call(Object event) {
       
               if(event instanceof TapEvent) {
                 _showTapText();
       
               }else if(event instanceof SomeOtherEvent) {
                 _doSomethingElse();
               }
             }
           });
   
   ```
   
## Operator

   操作符实际是在`源Observable` 和 `终Subscriber` 之间对数据源做各种变化操作,满足`终Subscriber` 的数据需求   
   如果不用操作符,就会将很多冗杂的代码放到 `Subscriber`的回调中处理,而理想的状态是`Subscriber`得到的最终数据类型
   是可以直接使用而不做任何的数据变化等
   
   The `Observable` and `Subscriber` are independent of the transformational steps in between them.
   
   ```java
   
      //map操作符可以将Source Observable发送的数据item转变成自己想要的任何类型
       Observable.just("Hello, world!")
       //第一个参数是Source Observable的数据类型,第二个参数是需要返回的数据类型
          .map(new Func1<String, String>() {
              @Override
              public String call(String s) {
                  return s + " Brucetoo";//在发射的字符串后 加上特定一个字符串
              }
          })
          .subscribe(s -> System.out.println(s)); 
          
          
      //flapMap介绍
      1.假设有一个query方法查询后返回一个列表
       Observable<List<String>> query(String text); 
      2.现在需要输出每个列表
        传统的方式:
        query("test")
            .subscribe(urls -> {
                //在subsribe中循环输出url
                for (String url : urls) {
                    System.out.println(url);
                }
            }); 
        flapMap使用的方式:flapMap最终返回一个Observable
        query("test")
            .flatMap(new Func1<List<String>, Observable<String>>() {
                @Override
                public Observable<String> call(List<String> urls) {
                    //返回的Observable是通过from操作符转换的
                    return Observable.from(urls);
                }
            })
            .subscribe(url -> System.out.println(url));
        

   ```
   
   `flatMap()` 和 `map()` 都将把传入的参数转化之后返回另一个对象.
   但`flatMap()` 中返回的是个 `Observable` 对象,
   并且这个 `Observable` 对象**并不是被直接**发送到了 `Subscriber` 的回调方法中.
   `flatMap()` 的原理是这样的: [选自](http://gank.io/post/560e15be2dca930e00da1083#toc_1)
   - 使用传入的事件对象创建一个 `Observable` 对象;
   - 并不发送这个 `Observable`,而是将它激活，于是它开始发送事件;
   - 每一个创建出来的 `Observable` 发送的事件,都被汇入同一个 `Observable`,而这个 `Observable` 负责将这些事件统一交给 `Subscriber` 的回调方法。
    其实就是通过一组新创建的 `Observable` 将初始的对象『铺平』之后通过统一路径分发了下去。而这个『铺平』就是 `flatMap()` 所谓的 `flat`
      
