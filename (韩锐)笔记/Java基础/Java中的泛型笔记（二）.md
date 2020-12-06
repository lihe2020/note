似乎是为向后兼容（不太了解历史，呵呵），Java虚拟机没有泛型类型，所有的泛型类最终会被编译为普通类。  
类型参数将在编译期间被替换成相应的原始类型，具体来说：如果泛型类型参数没有限定（约束），那么类型参数将会被替换成`Object`，
否则使用第一个限定类型替换，这就是泛型类型擦除(erased)。例如：
``` java
public class Pair<T> {
    private T first;
    private T second;

    public Pair(T first, T second) {
        this.first = first;
        this.second = second;
    }

    public T getFirst() { return first; }
    public T getSecond() { return second; }
    public void setSecond(T second) { this.second = second; }
}
```
因为`T`是没有限定的泛型参数，所以`Pair<T>`中所有的`T`将会被替换成`Object`，那么`Pair<T>`看起来将会是：
``` java
public class Pair {
    private Object first;
    private Object second;

    public Pair(Object first, Object second) {
        this.first = first;
        this.second = second;
    }

    public Object getFirst() { return first; }
    public Object getSecond() { return second; }
    public void setSecond(Object second) { this.second = second; }
}
```
如果`T`有`Serializable`约束，即`class Pair<T extends Serializable> { ... }`，那么`T`将会被替换成`Serializable`。

## 泛型表达式的类型擦除机制
当程序调用泛型方法时，如果擦除返回类型，编译器将插入强制类型转换。例如：
``` java
Pair<String> pair = new Pair<>("foo", "bar");
String first = pair.getFirst();
```
由于`getFirst`的返回类型被擦除成`Object`类型，为了能让这段代码正常工作，编译器将把这个方法调用翻译成2条虚拟机指令：
1. 对原始方法`Pair.getFirst`的调用。
2. 将返回的`Object`类型强制转换成`String`类型。

方法的擦除在类继承层次中带来一个问题，考虑到下面的示例：
``` java
public class DateInterval extends Pair<Date> {
    //...

    @Override
    public void setSecond(Date second) {
        super.setSecond(second);
    }
}
```
按照正常思维，我们认为`setSecond`方法重写类超类相同方法，也就是`setSecond`应该具有多态性，下面的调用：
``` java
Pair<Date> pair = new DateInterval(...);
pair.setSecond(...);
```
调用`pair.setSecond`应该调用的是`DateInterval`的方法。但，由于有类型擦除机制，超类的方法其实是：
``` java
public void setSecond(Object second) { this.second = second; }
```
由于2个方法的签名不一样，不具有多态性。编译器为了解决这个问题，在`DateInterval`类中生成类一个**桥方法**(bridge method)：
``` java
public void setSecond(Object second) { this.setSecond((Date)second); }
```
如此，对`pair.setSecond`的调用，会先调用桥方法（因为重写类超类相同的方法，所以它具有多态性），再由桥方法调用`DateInterval`的方法，这正式我们期待的效果。


## 约束与局限性

由于Java中的泛型使用的是类型擦除机制，而并非真正的泛型，这种伪泛型导致它有很多的限制，下面一一列举：

### 不能使用基元（基本）类型
由于类型擦除，(没有约束的)泛型类型最终会被替换成`Object`，而它不能用于存储`int`、`double`等基元类型。所以如果需要使用这些类型，需要将它们装箱(int -> Integer)。

### 泛型类型不能用于类型检测
下面的代码都将编译出错：
``` java
if(a instanceof Pair<String>) //error
if(a instanceof Pair<T>) //error
```
因为类型擦除后只有`Pair`类型。同样，下面的判断将成立：
``` java
Pair<String> sp = ...
Pair<Employee> se = ...
if(sp.getClass() == se.getClass()){
    System.out.println("the are equal");
}
```

### 不能创建泛型数组
这个限制是为了类型安全，因为向正常数组中添加无关的元素类型，运行时将会抛出`ArrayStoreException`，但泛型数组由于类型擦除，无法检测到这种问题，所以干脆不允许创建泛型数组。具体来说：
``` java
String[] table = new String[10];
Object[] objarray = table;
objarray[0] = new Employee(); //error, ArrayStoreException
```
向String类型的数组中添加Employee类型，这不故意找茬嘛，当然是不允许的。而对于泛型数组来说，类型擦除后将是`Object[]`类型，所以
它将允许添加任意类型，因为Object是所有类型的超类，这不是我们想要的，但这种情况没办法检测到，所以为了类型安全，申明泛型数组将出错。  
但（峰回路转），虽然不允许泛型数组，但可以传递它，考虑到如下示例：
``` java
@SafeVarargs
public static <T> void addAll(Collection<T> coll, T... ts){
    //...
}
```
参数`ts`正是一个泛型数组，这是允许的，但编译器将会给出一个警告，因为它是不安全的。为了抑制这个警告，需要使用`@SafeVarargs`注解来标注该方法。


### 不能创建类型实例
同样由于类型擦除（或者没有像C#那么样的`new()`泛型约束？），下面的调用将编译出错：
``` java
T instance = new T(); //error
Class<T> = T.class; //error
```
正因为如此，下面的泛型方法没有任何参数，它没办法创建`T`的实例：
``` java
public static <T> Pair<T> makePair(){
    //此处无法创建T的实例
}
```
但，下面这种情况是可以创建实例的：
``` java
public static <T> Pair<T> makePair(T defaultValue) {
    try {
        T instance = (T)defaultValue.getClass().newInstance();
        return new Pair<>(instance, instance);
    }catch (Exception e){
        return null;
    }
}
 ```
 或者：
 ``` java
public static <T> Pair<T> makePair(Class<T> cl) {
    ...
    T instance = cl.newInstance();
    ...
 }
 ```


 ### 其他
 其他包括不能创建静态泛型类、不能抛出或补货泛型类实例等，具体参见《Java核心技术 卷1》。

 ## 总结
 。。。

