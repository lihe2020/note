内部类，也称为嵌套类(Nested Classes)，一个被嵌套的类包含在外围类的作用域内。嵌套是一种类之间的关系，而不是对象之间的关系。这种类有2个好处：命名控制和访问权限。  

在Java中，内部类的功能得到了增强：内部类的对象有一个隐式引用，它引用了实例化该内部对象的外围类对象。通过这个引用，可以访问外围类对象的全部状态。当然，如果不需要这些增强，可以定义为静态内部类(static inner classes)，它与嵌套类很相似。  

## 内部类的特殊语法规则
在内部类中引用外围类使用语法`OuterClass.this`。初始化内部类使用语法`outerObject.new InnerClass(...)`，如果在外围类中初始化，那么`outerObject`用`this`替代，例如：
``` java
import java.time.LocalTime;

public class TimePrinter {
    private LocalTime time;

    public TimePrinter() {
        this.time = LocalTime.now();
    }

    public void printTime(){
        //初始化内部类，outerObject为this
        Printer printer = this.new Printer();
        printer.print();
    }

    class Printer{
        public void print(){
            //TimePrinter.this.可省略
            System.out.printf("现在时间是：%s.", TimePrinter.this.time);
        }
    }
}
```
如果内部类的作用域为public或包可见（没有定义任何限定符），那么可以通过如下方式初始化和访问内部类：
``` java
public class TimePrinterTest {
    public static void main(String... args){
        TimePrinter timerPrinter = new TimePrinter();
        //outerObject为timerPrinter
        TimePrinter.Printer printer = timerPrinter.new Printer();
        printer.print();
    }
}
```

## 内部类的原理
内部类是一种编译器现象，与虚拟机无关。编译器将会把内部类翻译成用$符号分隔外部类名与内部类名的常规类文件，而虚拟机则对此一无所知。例如，上面的内部类`Printer`将会被编译成`TimePrinter\$Printer.class`。那既然如此，会产生3个问题：

1. 内部类与外围类如何关联起来的？
2. 内部类如何访问外围类的私有成员？
3.  ~~常规类没有private限定符，内部类是如何解决私有类的？~~ 
为了揭开这些谜团，可以使用javap查看编译后的文件：
``` bash
#编译并产生.class文件
javac TimePrinterTest.java
#反编译，\$用于转义$
javap -private TimePrinter TimePrinter\$Printer
```
输出如下：
``` java
Compiled from "TimePrinter.java"
public class TimePrinter {
  private java.time.LocalTime time;
  public TimePrinter();
  public void printTime();
  static java.time.LocalTime access$000(TimePrinter);
}
Compiled from "TimePrinter.java"
class TimePrinter$Printer {
  final TimePrinter this$0;
  TimePrinter$Printer(TimePrinter);
  public void print();
}
```
首先，可以看到`TimePrinter$Printer`类多了一个`this$0`字段，且构造函数多了一个参数，这是编译器为了引用外围类才这么做的，
因此问题1也就解决了。
对于问题2，可以看到编译器为外围类`TimePrinter`添加了一个静态方法`access$000`，它的返回值正是内部类引用的那个（外围类）的私有字段，而实际上，内部类正是通过调用该方法来访问私有成员的。

## 匿名内部类
很多时候，内部类只会在一个地方（方法）内使用，这个时候可以创建匿名局部类，而不必命名，通用的语法为：
``` java
new SuperType(construction parameters){
    //...
}
```
其中`SuperType`可以是接口，此时内部类需要实现这个接口。SuperType也可以是一个类，此时内部类扩展它。
由于构造函数的名字必须与类名相同，而匿名类没有类名，所以匿名类不能有构造函数。
另外，如果匿名内部类定义在方法内，并且需要访问方法内的局部变量，那么这些变量需要声明为`final`，因为编译器需要将这些局部变量传递到类构造函数中，以便将这些参数初始化为局部变量的副本。
下面的示例实现了在一个新的线程中打印通过参数传递的时间：

``` java
import java.time.LocalTime;

public class TimePrinterTest {
    public static void main(String... args){
        printTime(LocalTime.now());
    }

    private static void printTime(final LocalTime time){
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                System.out.printf("现在时间：%s.", time);
            }
        };
        new Thread(runnable).start();
    }

}
```
可以看到，runnable是一个实现了`Runnable`接口的匿名类，它还引用了外围方法的参数`time`，可以看到他被定义为final。

## 静态内部类
正如本文开头所述，如果仅仅是想把一个类隐藏在另一个类内部，并不需要引用外围类对象，那么可以将内部类声明为`static`，以便
取消产生的引用。当然，只有内部类才能声明为static。

## 结束
在学习类内部类和匿名内部类后，我（目前）认为它主要是解决不能通过简单的方式传递方法的问题（类似于C#中匿名委托的使用方式），
但这种使用方式还是非常啰嗦，并且也有些限制。好在Java 8中有了新的替代方案--lambda，awesome！