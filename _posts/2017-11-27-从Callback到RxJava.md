---
layout: default
title: "从Callback到RxJava"
date: 2017-11-27 22:35:00 0600
categories: Android

---
<!--more-->

# 从Callback到RxJava

## 回调机制

软件模块之间总是存在着一定的接口，从调用方式上，可以把他们分为三类：同步调用、回调和异步调用。

+ 同步调用

  是一种阻塞式调用，调用方要等待对象执行完毕才返回，它是一种单向调用。

+ 回调

  被调用方在接口被调用时也会调用对方的接口，它是一种双向调用。

+ 异步调用

  是一种类似消息或事件的机制，不过它的调用方向刚好相反，接口的服务在收到某种讯息或发生某种事件时，会主动通知客户方（即调用客户方的接口）。

回调和异步调用的关系非常紧密，通常我们使用回调来实现异步消息的注解，通过异步调用来实现消息的通知。同步调用是三者当中最简单的，而回调又常常是异步调用的基础。

**回调机制按照约定的接口暴露给外部使用者，为外部使用者提供数据，或要求外部使用者提供数据。**

## Callback

按照功能可以分为：数据提供型回调`Provider Callback`和数据消费型回调`Consumer Callback`。

### Provider Callback

数据提供型回调。对象通过该回调从外部读取数据。

```java
public interface ProviderCallback {
  void call(Object obj);
}
```
或
```java
public interface ProviderCallback {
  Object call();
}
```

### Consumer Callback

数据消费型回调。对象通过该回调向外部提供数据。
```java
public interface ConsumerCallback {
  void call(Object obj);
}
```

### 实例

```java
public class CallbackTest {
  public static void main(String[] args) {
    // 负责存储数据的容器
    Container container = new Container();
    container.inject(new ProviderCallback() {
      @Override
      public void call(Object obj) {
        Map<String, String> map = (Map<String, String>) obj;
        map.put("Key", "Value"); // 向容器内部添加数据
      }
    });
    container.inject(new ProviderCallback2() {
      @Override
      public Map<String, String> call() {
        Map<String, String> map = new HashMap<>();
        map.put("New Key", "New Value");
        return map;  // 直接替换内部所有数据
      }
    })
    container.offer(new ConsumerCallback() {
      @Override
      public void call(Object obj) {
        Map<String, String> map = (Map<String, String>) obj;
        // 读取数据
        for (Map.Entry<String, String> entry : map.entrySet()) {
          System.out.println("key-value = [" + entry.getKey() + "," + entry.getValue() + "]");
        }
      }
    })

  }

  public static class Container {
    private Map<String, String> map = new HashMap<>();
    public void inject(ProviderCallback callback) {
      callback.call(map);
    }
    public void inject(ProviderCallback2 callback) {
      map = callback.call();
    }
    public void offer(ConsumerCallback callback) {
      callback.call(map);
    }
  }

  // 数据提供型回调
  public interface ProviderCallback {
    void call(Map<String, String> map);
  }
  // 数据提供型回调
  public interface ProviderCallback2 {
    Map<String, String> call();
  }
  // 数据消费型回调
  public interface ConsumerCallback {
    void call(Map<String, String> map);
  }
}
```

## 利用Callback实现数据传递

因为`Callback`既可以提供数据，又可以消费数据。那么如果我们将这样两种`Callback`进行串联，就可以形成一端输入数据一端输出数据的`数据通路`。

```java
public class DataChannel {

  private Map<String, String> mDataMap = new HashMap<>();
  public void inject(ProviderCallback callback) {
    callback.call(mDataMap);
  }

  public void offer(ConsumerCallback callback) {
    callback.call(mDataMap);
  }

}
```

我们可以通过`inject`方法向`DataChannel`添加数据，调用`offer`方法从`DataChannel`读取数据。

但是在这里有几点问题：限制了数据的类型；需要将数据临时存储到`DataChannel`内部。

如果我们可以直接向将数据输入端和数据输出端连接起来，`DataChannel`类仅负责端与端的连接，这样相对于上面的实现方式更加通用和高效。换个角度，相对于数据输入端，数据输出端本身就是一种数据容器，输入端可以直接将数据读取到输出端。这样可以由数据输入端和数据输出端共同约定数据的格式，而与`DataChannel`无关。

```java
public class DataChannel2<T> {
  public interface Provider<T> {
    void call(Consumer<T> consumer);
  }

  public interface Consumer<T> {
    void consume(T obj);
  }

  public static <T> DataChannel2<T> create(Provider<T> provider) {
    return new DataChannel2<>(provider);
  }

  private Provider<T> provider;

  protected DataChannel2(Provider<T> provider) {
    this.provider = provider;
  }

  public void consume(Consumer<T> consumer) {
    provider.call(consumer);
  }

  public static void main(String[] args) {
    DataChannel2
        .create(new Provider<String>() {
          @Override
          public void call(Consumer<String> consumer) {
            consumer.consume("Data");
          }
        })
        .consume(new Consumer<String>() {
          @Override
          public void consume(String s) {
            System.out.println("Receive: " + s);
          }
        });
  }
}
```

至此我们就实现了一个简单的类RxJava框架。

## RxJava

上文中的`DataChannel2`建立的模型是数据输入端将数据直接输出到数据输出端。

而`RxJava`建立的模型是相对比较复杂的模型。主要可以描述为：主题`Observable`对象在被订阅者`Observer`订阅时，会通过`ObservableOnSubscribe`通知主题`Observable`中的数据发射器`ObservableEmitter`发射数据，通过一系列处理后传递给订阅者。

```java
public class RxJavaDemo {
  public static void main(String[] args) {
    Observable
        .create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> e) throws Exception {
              try {
                  e.onNext("data1");
                  e.onNext("data2");
                  e.onComplete();
              } catch (Exception e1) {
                  e.onError(e1);
              }
            }
        })
        // ... 
        .subscribe(new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {
            }

            @Override
            public void onNext(String s) {
            }

            @Override
            public void onError(Throwable e) {
            }

            @Override
            public void onComplete() {
            }
        });
  }
}
```

+ Observable / ObservableSource

  主题接口

+ ObservableEmitter

  主题的数据发射器，主要用于发射数据

+ ObservableOnSubscribe

  主题触发数据源发送数据的接口。

+ Observer

  订阅者接口
