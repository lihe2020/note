由于公司的原因，本人需要开始学习java。
初入java，感觉java的世界不像.net/c#那么简洁、那么井然有序，有些技术之前有所耳闻，有些不明其意，另外一些从未听过。
此篇文章记录一些常见的名词和概念，以便随时翻阅。

## 基本术语
- **JRE**(Java Runtime Environment) 运行Java程序的软件
- **JVM**(Java Virtual Machine) JRE的一部分，用于编译和执行执行字节码
- **JDK**(Java Development Kit) 完整的开发套件，包括JRE、编译器等
三者的区别参见：https://www.javatpoint.com/difference-between-jdk-jre-and-jvm

- **OpenJDK** 它是Java SE 7 JSR的开源实现。现在它与Oracle JDK几乎一致。

## Java版本号
Java2，即第二代Java平台，包括3个版本：J2ME，主要用于嵌入式开发；J2SE，应用于桌面环境；J2EE，提供了企业级开发的完整解决方案，一般应用于服务端开发。 同时，JDK版本号经历了1.2、1.3、1.4、5.0。此后，Java 2命名被遗弃，各版本取消了其中的数字2，比如J2EE更名为Java EE。

## Bean相关
- **Java Bean** JavaBean包括以下特征：1、有拥有默认无参构造函数；2、提供getter/setter属性；3、相关类可以被序列化。
- **POJO**(Plain Old Java Object) 没有继承关系(extends/implements)的普通Java类
- **DTO**(Data Transfer Object) 数据传输对象，是一种设计模式之间传输数据的软件应用系统。
具体的比较参见：https://stackoverflow.com/a/1612671