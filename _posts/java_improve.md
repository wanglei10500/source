---
title: Java改善代码质量
tags:
 - java
categories: 经验分享
---
#### 显示声明UID
类实现Serializable接口的目的是为了可持久化，比如网络传输或本地存储，为系统的分布和异构部署提供先决条件支持。
```java
//类的序列化和反序列化方法
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.io.Serializable;

public class SerializationUtils {
    private static String FILE_NAME = "c:/obj.bin";
    //序列化
    public static void writeObject(Serializable s) {
        try {
            ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(FILE_NAME));
            oos.writeObject(s);
            oos.close();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    //反序列化
    public static Object readObject() {
        Object obj = null;
        try {
            ObjectInputStream input = new ObjectInputStream(new FileInputStream(FILE_NAME));
            obj=input.readObject();
            input.close();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        return obj;
    }
}
```
在类不一致时，反序列化会抛出InalidClassException异常，原因是序列化和反序列化对应的类版本不一致。

JVM通过serialVersionUID判断类版本，显式声明可以骗过JVM，带来的问题是消费者读不到类新增的属性，好处是可以实现版本向上兼容，提高健壮性。
```java
private static final long serialVersionUID = 1867341609628930239L;
```
#### 避免用序列化类在构造函数中为final修饰的赋值
反序列化时构造函数不会执行

反序列化的执行过程：JVM从数据流中获取一个Object对象，然后根据数据流中的类文件描述信息(在序列化时，保存到磁盘的对象文件中包含了类描述信息，注意是描述信息，不是类)查看，发现是final变量，需要重新计算，JVM发现final变量没有赋值，不能引用，于是它不再初始化，保持原值状态。
