异步编程，是并发的一种形式，它采用future模式或回调（callback）机制，以避免产生不必要的线程。
一个future类型代表一些即将完成的操作。异步编程的核心理念是异步操作：启动了的操作将会在一段时间后完成。这个操作正在执行时，不会阻塞原来的线程。启动了这个操作的线程，可以继续执行其他任务。当操作完成时，它会通知future，或者调用回调结束，以便让程序知道操作已经结束。
<br/>
在Java中，future类型有`Future`和`CompletableFuture`。其中`CompletableFuture`是在Java8中引入的，它是对`Future`的改进和补充。为了展示`CompletableFuture`强大等特性，我们将由浅入深介绍各个功能。
<br/>

## 1. 使用CompletableFuture创建异步任务

首先，由于`CompletableFuture`实现了`Future`接口，因此您可以将其用作Future实现，但需要一些额外的完成逻辑。
``` java
public Future<String> calculateAsync() {
    CompletableFuture<String> completableFuture = new CompletableFuture<>();
    new Thread(() -> {
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        completableFuture.complete("Hello");
    }).start();
    return completableFuture;
}
```
在这段代码中，首先创建了一个代表异步计算的CompletableFuture对象实例，它在计算完成时会包含计算结果。接着，使用一个新的线程去执行实际的工作，同时立即返回Future实例。在将来某个时刻计算工作完成时，使用它的complete方法，结束completableFuture对象的运行，并设置运行结果。  
<br/>
为了获取异步计算的结果，我们需要调用Future对象的get方法，它将同步阻塞直到异步任务完成：

``` java
Future<String> completableFuture = calculateAsync();
//...
String result = completableFuture.get();
```
一个值得注意的地方是，get方法会抛出一些已检测的异常，包括`InterruptedException`（线程中断异常）和`ExecutionException`（异步计算异常）。
<br/><br/>

对于异步计算，除了使用它的complete方法完成CompletableFuture外，还可以使用cancel方法取消它，该方法接收一个boolean类型参数mayInterruptIfRunning参数，这个参数仅用于兼容而不起作用：

``` java
public Future<String> calculateAsyncWithCancellation() {
    CompletableFuture<String> completableFuture = new CompletableFuture<>();
    new Thread(() -> {
        try {
            Thread.sleep(500);
            //cancel方法接收
            completableFuture.cancel(false);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }).start();
    return completableFuture;
}
```
这将导致在调用get方法获取结果时抛出`CancellationException`异常。
 
## 2. 使用工厂方法supplyAsync、runAsync创建CompletableFuture

上面例子我们通过CompletableFuture创建了异步计算任务，但它显得有些冗长。为此，CompletableFuture提供了工厂方法`supplyAsync`和`runAsync`来简化代码。
<br/><br/>

- `supplyAsync`方法接收一个Supplier类型参数，Supplier类型有一个返回值。
- `runAsync`方法接收一个Runnable类型参数，Runnable类型没有返回值。

这2个方法均接收的参数类型均函数式接口类，上面例子可以简化为：

``` java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "Hello");
```

这里创建的异步任务会交由ForkJoinPool线程池的某个Executor来处理。当然，您也可以使用supplyAsync的重载版本，使用不同的Executor来处理。

## 3. 使用thenApply和thenAccept方法转换结果

在异步任务执行完后，您可能需要进行一些后续处理，将异步结果转换成合适的类型，相当于将类型`CompletableFuture<T>`转换成`CompletableFuture<U>`。为此，您需要使用`thenApply`方法，它接收`Function`类型参数，
并返回一个新的、包装了`Function`返回值的CompletableFuture。

``` java
CompletableFuture<String> future
    = CompletableFuture.supplyAsync(() -> "Hello")
        .thenApply(s -> s + " World");
 
assertEquals("Hello World", future.get());
```

<br/>
如果您的后续处理不需要返回值，那么可以使用`thenAccept`，它接收`Consumer`类型参数；如果您的后续处理甚至不关心前一个任务的返回值，那么可以使用`thenRun`，它接收`Runnable`类型参数。这些方法的签名如下：

``` java
CompletableFuture<U> thenApply(Function<? super T,? extends U> fn);
CompletableFuture<Void> thenAccept(Consumer<? super T> action);
CompletableFuture<Void> thenRun(Runnable action);
```

具体用法请参考文档。

## 4. 使用thenCombine和thenCompose方法组合CompletableFuture

CompletableFuture API最值得一提的地方是能够将多个CompletableFuture组合成一条流水线任务。这种方法在函数式语言中无处不在，通常被称为monadic设计模式。
<br/>
<br/>

假如，您有一个根据订单Id查询订单的异步方法`getOrderAsync`，和一个根据物流Id查询物流的异步方法`getDeliveryAsync`。现在您需要根据订单Id查询订单信息，然后根据订单实体中的物流Id查询
物流信息。此时，需要使用`thenCompose`方法来将这些步骤组合成流水线：

``` java
public CompletableFuture<Order> getOrderAsync(long orderId){
    //...
}
public CompletableFuture<Delivery> getDeliveryAsync(long deliveryId){
    //...
}

CompletableFuture<Delivery> completableFuture
    = getOrderAsync(123456)
        .thenApply(order -> order.getDeliveryId())
        .thenCompose(deliveryId -> getDeliveryAsync(deliveryId));

```

`thenCompose`方法将两个异步操作组合成流水线，第一个操作完成时，将其结果作为参数传递给第二个操作。
<br/><br/>

与`thenCompose`类似，`thenCombine`方法也用于组合2个CompletableFuture，但区别是`thenCombine`用于组合2个没有关联的CompletableFuture。
<br/><br/>

假如，您有一个根据用户Id查询用户的异步方法`getUserAsync`，和一个根据用户Id查询最新订单的异步方法`getNewestOrderAsync`。现在您需要提供一个接口，根据用户Id同时查询用户数据和最新订单数据：
``` java
private CompletableFuture<User> getUserAsync(long userId){
    //...
}
private CompletableFuture<Order> getNewestOrderAsync(long userId){
    //...
}

long userId = 121212;
CompletableFuture<UserWithOrder> completableFuture
        = getUserAsync(userId)
            .thenCombine(
                    getNewestOrderAsync(userId), (user, order) -> new UserWithOrder(user, order)
            );

```

`thenCombine`方法组合的2个异步操作是同时执行的，并且它提供了如何合并两个CompletableFuture对象完成结果。<br/>

与`thenCombine`类似的另一个方法`thenAcceptBoth`，它同样用于组合2个异步CompletableFuture，区别是它没有返回值。
<br/><br/>

以下是这些方法的主要签名：

``` java
CompletableFuture<U> thenCompose(Function<? super T, ? extends CompletionStage<U>> fn);
CompletableFuture<V> thenCombine(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn);
CompletableFuture<Void> thenAcceptBoth(CompletionStage<? extends U> other, BiConsumer<? super T, ? super U> action);
```

## 5. 并行执行多个Futures

当我们需要并行执行多个操作时，我们通常希望等待所有这些操作执行完成并处理它们的结果。`CompletableFuture.allOf`静态方法允许等待所有操作完成；
类似的，`CompletableFuture.anyOf`静态方法允许等待任意操作完成。
这2个方法的签名如下：

``` java
static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs);
static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs);
```

## 6. 异步方法

CompletableFuture API中大多数方法提供了Async结尾的版本，如：`thenApplyAsync`、`thenComposeAsync`等，这些方法通常用于在另一个线程中执行相应的操作。
通常而言，方法名称中不带Async的方法和它的前一个任务一样，在同一个线程中运行；而名称以Async结尾的方法会将后续的任务提交到一个线程池，每个任务是由不同的线程处理的。

## 7. 异常处理

在异步编程中使用try/catch来处理异常显然是不够优雅的，为此，CompletableFuture提供了多个与异常有关的方法，包括：

``` java
CompletableFuture<U> handle(BiFunction<? super T, Throwable, ? extends U> fn);
CompletableFuture<T> exceptionally(Function<Throwable, ? extends T> fn);
CompletableFuture<T> whenComplete(BiConsumer<? super T, ? super Throwable> action);
```

这3个方法的区别：
- `handle` 用于处理前一个任务的结果，如果前一个任务有异常，那么fn中的异常参数将不为null，否则结果参数不为null
- `exceptionally` 用于处理前一个任务的异常，它仅当前一个任务有异常时才会执行
- `whenComplete` 在前一个任务完成后执行，它不能用于处理前一个任务的结果。也就是说如果前一个任务包含异常，whenComplete执行完后返回的CompletableFuture仍然包含异常。

<br/>

另一个需要注意的地方是，这3个返回都返回一个新的`CompletableFuture`。如果在执行这3个方法时抛出了异常，那么handle、exceptionally返回的`CompletableFuture`将包含这个异常信息。
而whenComplete将分为2种情况：
1. 前一个任务没有异常，那么返回的`CompletableFuture`将包含当前产生的异常，这与前面2个方法一致
2. 前一个任务有异常，那么返回的`CompletableFuture`将包含前一个任务的异常

## 8. 总结

本文介绍了多个方法，下面进行总结：

1. 使用工厂方法创建CompletableFuture
   1. supplyAsync 接收Supplier类型参数，Supplier有一个返回值
   2. runAsync    接收Runnable类型参数，Runnable没有返回值
2. 转换结果
   1. thenApply  接收Function类型参数，Function有一个返回值
   2. thenAccept 接收Consumer类型参数，Consumer没有返回值
   3. thenRun    接收Runnable类型参数，Runnable没有返回值
3. 组合CompletableFuture
   1. thenCombine 将两个异步操作组合成流水线
   2. thenCompose 组合2个没有关联的CompletableFuture
   3. thenAcceptBoth 与thenCombine类似，但它没有返回值
4. 并行执行
   1. allOf 等待所有操作完成
   2. anyOf 等待任意操作完成
5. 异常处理
   1. handle 处理前一任务结果
   2. exceptionally 处理前一任务异常
   3. whenComplete 在前一任务结束后执行，它不能处理前一步结果

## 9. 参考源码
http://gitlab.bitautotech.com/hanrui3/completablefuture-demo

## 10. 参考链接
[Guide To CompletableFuture](http://www.baeldung.com/java-completablefuture)  
[JAVA'S COMPLETEABLEFUTURE EXCEPTION HANDLING: WHENCOMPLETE VS. HANDLE](https://dempkow.ski/blog/java-completablefuture-exception-handling/)  
[Java CompletableFuture 详解](http://colobu.com/2016/02/29/Java-CompletableFuture)
