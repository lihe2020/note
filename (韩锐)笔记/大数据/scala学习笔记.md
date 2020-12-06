### 1. Scala数据类型

- Byte
- Short
- Int
- Long
- Float
- Double
- Char
- String
- Boolean
- Unit 表示无值，和void等同
- Null 空值或空引用
- Nothing 所有其他类型的子类型，表示没有值
- Any 所有类型的超类，任何实例都属于Any类型
- AnyRef 所有引用类型的超类
- AnyVal 所有值类型的超类

#### Scala中的空值

- `Null` Trait，其唯一实例为null，是AnyRef的子类
- `Nothing` Trait，所有类型的子类，没有实例
- `None` Option的两个子类之一，另一个是Some，用于安全的函数返回值
- `Unit` 无返回值的函数的类型，和java的void对应
- `Nil `长度为0的List

### 2. 常用语法

- 简单语法

  ```scala
  object entry {
    def main(args: Array[String]): Unit = {
      // 不可变变量
      val a = 123;
      // 可变变量
      var b = 123;
  
      // for循环，其中to叫做操作符
      for (i <- 1 to 10) {
        // s前缀表示是字符串表达式
        print(s"loop $i")
      }
      
      // 定义一个匿名类型
      def anonymousFunc(n Int) = {
        n
      }
      
      // 偏应用函数
      
      // 高阶函数
      
    }
  }
  ```
  
- object定义的是静态类，类中的方法都是静态的

- 类定义的一些语法

  ```scala
  // 1. 定义Person类和默认构造函数
  class Person(xname: String){
    
    // 2. 定义属性，缺省的限定符是public
    val name = xname
    private var age : Int = 0
  
    // 3. 定义非默认构造函数
    def this(xname: String, xage: Int) {
      this(xname)
      this.age = xage
    } 
    
    // 4. 定义一个方法
    def sayHi(): Unit = {
      println(s"hello, $name, age $age.")
    }
    
    // 5. 如果不写等号，那么方法被认为返回值是Unit
    def doSomething() {
      // 此时方法的返回值会被丢弃
      "nothing"
    }
    
    // 6. 如果方法只有一行代码，那么可以省略大括号
    // 7. return语句可以省略，此时也可以省略返回值，会自动推断
    def max(a: Int, b: Int) = if (a > b) a else b
    
  
  }
  ```

  ### 3. 集合语法

  ```scala
  // 创建不可变数组方式一
  val arr1 = Array[Int](1, 2, 3)
  // 创建不可变数组方式二
  val arr2 = new Array[Int](5)
  // 创建不可变数组方式三
  val arr3 = Array.fill(5)("test")
  // 创建可变数组方式
  import scala.collection.mutable.ArrayBuffer
  val arr4 = ArrayBuffer[Int](1,2,3)
  
  // 读取/设置数组元素
  arr1(1) = 0
  
  // 数组遍历
  arr1.foreach(println)
  
  // 追加元素（针对可变数组）
  // 向后追加
  arr4.+=(4)
  arr4.append(5)
  // 向前追加
  arr4.+=:(0)
  
  
  // 定义不可变list
  val list = List[Int](1, 2, 3)
  // 定义可变list
  import scala.collection.mutable.ListBuffer
  val list2 = new ListBuffer[Int]()
  // 不可变转成可变
  
  // 可变转成不可变
  val list4 = list2.toList
  
  
  // 定义不可变set(无序)
  val set1 = Set[Int](1, 2, 3)
  val set2 = Set[Int](3, 4, 5)
  
  // 求交集和差集
  val result1 = set1 & set2 // 交集，同set1.intersect()
  val result2 = set1 &~ set2 // 差集，同set1.diff()
  
  // 定义不可变map方式一
  val map1 = Map[Int, String](1 -> "a", 2 -> "b")
  // 定义不可变map方式二
  val map2 = Map[Int, String]((1, "a"), (2, "b"), (3, "c"))
  // 定义可变map
  val mmap = mutable.Map[Int,String]()
  // 读取元素
  val value: Option[String] = map1.get(1)
  // 新增元素（可变集合）方式一
  mmap(1) = "a"
  // 新增元素（可变集合）方式二
  mmap += (1 -> "a")
  // 移除元素（可变集合）方式一
  mmap -= 1
  // 合并map
  val combined = map1.++(map2)
  ```

  ### 4. 元组类型

  ```scala
  // 元组
  val tuple1 = new Tuple1[String]("hello")
  val tuple2 = Tuple2("hello", "world")

  // 读取第一个元素
  val hello = tuple1._1

  // 遍历
  val iter = tuple2.productIterator
  iter.foreach(println)
  ```
  
  ### 5. trait类型
  
  trait和java的接口很像，定义的方法可以实现，也可以不实现。
  
  ```scala
  trait Test {
  }
  
  trait IsEquals {
    // 定义方法
    def isEquals(o: Any): Boolean
  
    // 定义方法并实现
    def isNotEquals(o: Any): Boolean = !isEquals()
  }
  
  // 继承多个trait用with关键字
  class Point extends IsEquals with Test {
    val x: Int = 0
    val y: Int = 0
  
    override def isEquals(o: Any): Boolean = {
      if (!o.isInstanceOf[Point]) {
        false
      } else {
        val p = o.asInstanceOf[Point]
        p.x == this.x && p.y == this.y
      }
    }
  }
  ```

  ### 6. 模式匹配

  ```scala
  // 模式匹配
  def matchTest(o: Any) = {
    o match {
      case 1 => println("value is 1")
      case i: Int => println(s"type is Int, value is $i")
      case _ => println("match nothing")
    }
  }
  
  // 偏函数，只能匹配一个值，匹配上了返回某个值
  def ptest: PartialFunction[String, Int] = {
    case "a" => 1
    case "abc" => 3
    case _ => 0
  }
  ```

  ### 7. 样例类（case classes）

  ```scala
  // 样例类具有以下特点：
  // 1. 通过构造函数定义的参数会自动生成相应属性(getter、setter)
  // 2. 样例类可以new，也可以不用new
  // 3. 会自动重写toString, equals, copy和hashCode等
  case class Person(var name:String, var age: Int)
  ```

  ### 8. 隐式值与隐式参数

  1. 隐式值
  
     ```scala
     // implicit修饰的参数叫隐式参数
     // 隐式参数必须要单独放在括号里
     def sayHi(name: String)(implicit age: Int): Unit ={
       println(s"hi, $name")
     }
     
     def main(args: Array[String]): Unit = {
       // 隐式值
       implicit val age: Int = 30
       // 会自动找隐式值，并作为参数调用
       sayHi("jeff")
     }
     ```
  
  2. 隐式方法
  
     ```scala
     class People {
       def canWalk(): Boolean = true
     }
     
     case class Jeff(name: String)
     
     object entry {
     
       // 隐式转换函数，会自动将Jeff类型转换从People类型
       implicit def jeffToPeople(jeff: Jeff): People =    {
         new People()
       }
     
       def main(args: Array[String]): Unit = {
         val jeff = Jeff("jeff")
         // canWalk方法定义在People类中
         // 所以jeff会被隐式转换成People对象
         jeff.canWalk()
       }
     }
     ```
  
  3. 隐式类
  
     隐式类有点类似于C#的扩展方法，可以在类之外定义方法：
  
     ```scala
     object entry {
     
       implicit class ExtJeff(j: Jeff) {
         def showName(): Unit = {
           println(j.name)
         }
       }
     
       def main(args: Array[String]): Unit = {
         val jeff = Jeff("jeff")
         jeff.showName()
       }
     }
     ```

  ### 9. Actor

  

  ### 参考

  1. https://www.bilibili.com/video/BV1oJ411m7z3

