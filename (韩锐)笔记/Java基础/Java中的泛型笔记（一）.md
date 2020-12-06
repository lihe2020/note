在Java中，使用泛型可以避免强制类型转换，以及编译时类型检查，使程序具有更好的可读性和安全性。

## 泛型的基本语法规则

### 泛型类的语法规则
泛型类的定义需要用尖括号将类型变量括起来，放到类名后。泛型类可以有多个类型变量。例如：
``` java
public class Pair<T, U> {
    //...
}
```
在使用时，只需将类型变量替换为具体的类型即可：`ArrayList<String> files = new ArrayList<String>();`，此处构造函数的泛型类型可以省略：`ArrayList<String> files = new ArrayList<>();`，省略的类型可以从变量的类型推断得出。  
需要注意的是，通常使用E表示集合的元素类型，使用K和V表示字典的键、值，使用T（或U和S）表示任意类型。  

### 泛型方法的语法规则
泛型方法的定义需要将类型变量放在修饰符的后面，返回类型的前面：
``` java
public class Program {
    public static <T> T aMethod(T... a){
        return a[0];
    }
}
```
在使用时，需要将具体类型放在方面前，`Program.<String>aMethod("hello");`，此处由于编译器可以推断出泛型参数的类型，所以`<String>`可以省略。

### 类型变量的限定
有时候需要将类型变量限定为实现了特定接口、或继承了特定类的类型，可以使用`<T extends BoundingType>`语法，如果有多个类型变量需要限定，可以用逗号分隔，如果一个类型变量有多个限定，可以使用&分割，如：
``` java
class Pair<T extends Comparable & Serializable, U extends Comparable>{
    //...
}
```
## 通配符类型
根据泛型类型的继承规则，即使`class Manager extends Employee { ...}`，但`ArrayList<Manager>`和`ArrayList<Employee>`却没有任何联系，所以下面的代码将会编译出错：
``` java
ArrayList<Manager> managers = new ArrayList<>();
//Error:(40, 41) java: 不兼容的类型: java.util.ArrayList<Manager>无法转换为java.util.ArrayList<Employee>
ArrayList<Employee> employees = managers;
```
在Java中，可以使用通配符来解决这个问题。通配符共有3中类型，包括：无限定通配符(unbounded wildcard)、子类限定通配符(upper bound wildcard)、超类限定通配符(lower bound wildcard)。接下来将逐个尝试：
### 无限定通配符
无限定通配符类型使用`?`来替代具体的类型变量，它代表的是任意元素类型。将`employees`变量声明为`ArrayList<?>`，即：
``` java
ArrayList<?> employees = managers;
```
此时的`employees`称为未知的ArrayList(arraylist of unknow)，它的类型变量是任意类型。虽然这能解决上面的编译出错问题，
但由于限定过于宽泛，编译器无法知道具体的元素类型，所以拒绝向employees新增元素。唯一的例外是null，因为它可以代表任何类型。  

另一方面，可以读取employees的元素，因为即使元素类型未知，但它至少是一个`Object`类型：
``` java
//编译通过
for (Object item: employees){
    System.out.println(item);
}
//编译出错
employees.add(new Object());
```
### 子类限定通配符
由于无限定通配符在读取元素时，只能假定元素类型是Object，那么访问元素实际类型成员将是一件很困难的事。例如，上例的for循环中，
item申明为Object类型，能对它做的事很有限。为了应对这个问题，需要使用子类限定通配符。将上面示例的`employees`变量声明为`ArrayList<? extends Employee>`：
``` java
ArrayList<? extends Employee> employees = managers;
```
与无限定通配符类似，`?`在这里代表的是未知类型，但不同的是，这里的未知类型被限定为`Employee`的子类。
`ArrayList<? extends Employee>`表示任何ArrayList类型，它的类型参数是`Employee`的**子类**，包括`Employee`。
实际上，无限定通配符可以看作类型限定为Object子类的通配符，即：`<?>`等于`<? extends Object>`。
因此，它也有和无限定通配符一样的限制：**只能用于泛型对象读取，而不能向泛型对象写入。** 例如：
``` java
//编译正确
for (Employee item: employees){
    System.out.println(item);
}
//编译错误：提示参数不匹配
employees.add(new Employee());
```
### 超类限定通配符
为了能在使用了通配符的泛型对象中写入，需要使用另一种限定符：**超类**限定通配符。超类限定通配符与子类限定通配符相反：它限定为特定类的任意超类（基类）。例如下面方法：
``` java
public static void addManager(ArrayList<? super Manager> employees){
    //...
}
```
方法参数`employees`表示任何ArrayList类型，它的类型参数是`Employee`的超类或本身。所以可以这么调用该方法：
``` java
addEmployees(new ArrayList<Employee>());
addEmployees(new ArrayList<Object>());
```
在方法内部，由于`employees`的类型参数限定为`Manager`及其超类，所以可以确定一点：`Manager`及其子类和类型参数兼容，也就是
无论如何`Manager`及其子类可以隐式转换成`employees`的元素类型。因此，向`employees`集合中添加`Manager`是合法、安全的；反而，向集合中添加`Employee`是非法的，因为类型限定的参数不确定是否与`Employee`兼容：
``` java
public static void addManager(ArrayList<? super Manager> employees){
    employees.add(new Manager());
    //编译错误
    employees.add(new Employee());
}
```

**直观地讲，带有超类限定的通配符可以向泛型对象写入，带有子类型限定的通配符可以从泛型对象读取。**  
借用一个网上的复制泛型集合元素的[例子](https://segmentfault.com/a/1190000005337789)，直观的演示了这2中通配符的使用：
``` java
private static <E> void copy(List<? super E> dest, List<? extends E> source){
    for (E value : source){
        dest.add(value);
    }
}
```

## 参考
[https://docs.oracle.com/javase/tutorial/extra/generics/wildcards.html](https://docs.oracle.com/javase/tutorial/extra/generics/wildcards.html)  
[https://docs.oracle.com/javase/tutorial/extra/generics/morefun.html](https://docs.oracle.com/javase/tutorial/extra/generics/morefun.html)  
[http://www.angelikalanger.com/GenericsFAQ/FAQSections/TypeArguments.html#Wildcards](http://www.angelikalanger.com/GenericsFAQ/FAQSections/TypeArguments.html#Wildcards)  
[Java 泛型总结（三）：通配符的使用](https://segmentfault.com/a/1190000005337789)  
