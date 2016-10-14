---
title: Java改善代码质量
tags:
 - java
categories: 经验分享
---
### 显示声明UID
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
### 避免用序列化类对final进行复杂赋值
反序列化时构造函数不会执行

反序列化的执行过程：JVM从数据流中获取一个Object对象，然后根据数据流中的类文件描述信息(在序列化时，保存到磁盘的对象文件中包含了类描述信息)查看，发现是final变量，需要重新计算，JVM发现final变量没有赋值，不能引用，于是它不再初始化，保持原值状态。

保存到磁盘上(或网络传输)的对象文件包括两部分:

(1).类描述信息：包括类路径、继承关系、访问权限、变量描述、变量访问权限、方法签名、返回值、以及变量的关联类信息。要注意一点是，它并不是class文件的翻版，它不记录方法、构造函数、static变量等的具体实现。之所以类描述会被保存，很简单，是因为能去也能回嘛，这保证反序列化的健壮运行。

(2).非瞬态(transient关键字)和非静态(static关键字)的实体变量值

总结一下：反序列化时final变量在以下情况下不会被重新赋值:
* 通过构造函数为final变量赋值
* 通过方法返回值为final变量赋值
* final修饰的属性不是基本类型

### 使用序列化类私有方法部分属性序列化
```java
 private transient Salary salary;
 //序列化委托方法
private void writeObject(ObjectOutputStream oos) throws IOException {
    oos.defaultWriteObject();
    oos.writeInt(salary.getBasePay());
}
//反序列化委托方法
private void readObject(ObjectInputStream input)throws ClassNotFoundException, IOException {
    input.defaultReadObject();
    salary = new Salary(input.readInt(), 0);
}
```
java序列化回调：Java调用ObjectOutputStream类把一个对象转换成数据流时，会通过反射（Refection）检查被序列化的类是否有writeObject方法，并且检查其是否符合私有，无返回值的特性，若有，则会委托该方法进行对象序列化，若没有，则由ObjectOutputStream按照默认规则继续序列化。同样，在从流数据恢复成实例对象时，也会检查是否有一个私有的readObject方法，如果有，则会通过该方法读取属性值，此处有几个关键点需要说明：

* oos.defaultWriteObject():告知JVM按照默认的规则写入对象，惯例的写法是写在第一行。
* input.defaultReadObject():告知JVM按照默认规则读入对象，惯例的写法是写在第一行。
* oos.writeXX和input.readXX

### 避免instanceof非预期结果
null instanceof String：返回值为false，这是instanceof特有的规则，若做操作数为null，结果就直接返回false，不再运算右操作数是什么类。

### 不要只替换一个类
```java
public class Constant {
    public static final int MAX_AGE=180;
}
```
修改MAX_AGE时 对于final修饰的基本类型和String类型，编译器会认为它是稳定态的(Immutable Status)所以在编译时就直接把值编译到字节码中了，避免了在运行期引用（Run-time Reference），以提高代码的执行效率。

deploy时不要只替换这一个文件
### 不要让类型默默转换
```java
public static final int LIGHT_SPEED = 30 * 10000 * 1000;
long dis2 = LIGHT_SPEED * 60 * 8; // 会为负数值
```
注意：基本类型转换时，使用主动声明方式减少不必要的Bug.
```java
long dis2 = 1L * LIGHT_SPEED * 60L * 8
```
### 边界问题
```java
// 一个会员拥有产品的最多数量
public final static int LIMIT = 2000;
 // 会员当前用有的产品数量
int cur = 1000;
int order = input.nextInt();
if (order > 0 && order + cur <= LIMIT) {
      System.out.println("你已经成功预定：" + order + " 个产品");
    } else {
      System.out.println("超过限额，预定失败！");
  }
```
若输入2147483647会产生预订成功的错误。order的值是2147483647那再加上1000就超出int的范围了，其结果是-2147482649，原因：数字越界使校验条件失效。

相关边界问题应有边界测试，以下三个值是必须测试的:0、正最大、负最小

###  银行家舍入法则
普通的四舍五入对于一方有损失　0.000 + 0.001 + 0.002 + 0.003 + 0.004 - 0.005 - 0.004 - 0.003 - 0.002 - 0.001 = - 0.005；

银行家舍入(Banker's Round)的近似算法，其规则如下：
* 舍去位的数值小于5时，直接舍去；
* 舍去位的数值大于等于6时，进位后舍去；
* 当舍去位的数值等于5时，分两种情况：5后面还有其它数字(非0)，则进位后舍去；若5后面是0(即5是最后一个数字)，则根据5前一位数的奇偶性来判断是否需要进位，奇数进位，偶数舍去。

以上规则汇总成一句话：四舍六入五考虑，五后非零就进一，五后为零看奇偶，五前为偶应舍去，五前为奇要进一。

round(10.5551)  =  10.56   round(10.555)  =  10.56   round(10.545)  =  10.54
```java
import java.math.BigDecimal;
import java.math.RoundingMode;

public class Client25 {
    public static void main(String[] args) {
        // 存款
        BigDecimal d = new BigDecimal(888888);
        // 月利率，乘3计算季利率
        BigDecimal r = new BigDecimal(0.001875*3);
        //计算利息
        BigDecimal i =d.multiply(r).setScale(2,RoundingMode.HALF_EVEN);
        System.out.println("季利息是："+i);

    }
}
```
目前Java支持以下七种舍入方式：

* ROUND_UP：原理零方向舍入。向远离0的方向舍入，也就是说，向绝对值最大的方向舍入，只要舍弃位非0即进位。
* ROUND_DOWN：趋向0方向舍入。向0方向靠拢，也就是说，向绝对值最小的方向输入，注意：所有的位都舍弃，不存在进位情况。
* ROUND_CEILING：向正无穷方向舍入。向正最大方向靠拢，如果是正数，舍入行为类似于ROUND_UP；如果为负数，则舍入行为类似于ROUND_DOWN.注意：Math.round方法使用的即为此模式。
* ROUND_FLOOR：向负无穷方向舍入。向负无穷方向靠拢，如果是正数，则舍入行为类似ROUND_DOWN，如果是负数，舍入行为类似以ROUND_UP。
* HALF_UP：最近数字舍入(5舍)，这就是我们经典的四舍五入。
* HALF_DOWN：最近数字舍入(5舍)。在四舍五入中，5是进位的，在HALF_DOWN中却是舍弃不进位。
* HALF_EVEN：银行家算法

　　在普通的项目中舍入模式不会有太多影响，可以直接使用Math.round方法，但在大量与货币数字交互的项目中，一定要选择好近似的计算模式，尽量减少因算法不同而造成的损失。

注意：根据不同的场景，慎重选择不同的舍入模式，以提高项目的精准度，减少算法损失。

### 优先使用整型池
通过valueOf方法对int装箱成Integer，若整数在-128到127之间，会从IntegerCache的cache数组中获得，不在该范围的会new新的Integer对象

通过包装类型的valueOf生成的包装实例可以显著提高空间和时间性能。

### 不要覆写静态方法

```java
public class Client33 {
    public static void main(String[] args) {
        Base base = new Sub();
        //调用非静态方法
        base.doAnything();
        //调用静态方法
        base.doSomething();
    }
}

class Base {
    // 我是父类静态方法
    public static void doSomething() {
        System.out.println("我是父类静态方法");
    }

    // 父类非静态方法
    public void doAnything() {
        System.out.println("我是父类非静态方法");
    }
}

class Sub extends Base {
    // 子类同名、同参数的静态方法
    public static void doSomething() {
        System.out.println("我是子类静态方法");
    }

    // 覆写父类非静态方法
    @Override
    public void doAnything() {
        System.out.println("我是子类非静态方法");
    }
}
```
输出结果：   我是子类非静态方法      我是父类静态方法

实例对象有两个类型：表面类型(Apparent Type)和实际类型(Actual Type)，表面类型是声明的类型，实际类型是对象产生时的类型，比如我们例子，变量base的表面类型是Base，实际类型是Sub。对于非静态方法，它是根据对象的实际类型来执行的，也就是执行了Sub类中的doAnything方法。而对于静态方法来说，首先静态方法不依赖实例对象，它是通过类名来访问的；其次，可以通过对象访问静态方法，如果是通过对象访问静态方法，JVM则会通过对象的表面类型查找静态方法的入口.

在子类中构建与父类方法相同的方法名、输入参数、输出参数、访问权限(权限可以扩大)，并且父类，子类都是静态方法，此种行为叫做隐藏(Hide),它与覆写有两点不同：

1. 表现形式不同：隐藏用于静态方法，覆写用于非静态方法，在代码上的表现是@Override注解可用于覆写，不可用于隐藏。

2. 职责不同：隐藏的目的是为了抛弃父类的静态方法，重现子类方法，例如我们的例子，Sub.doSomething的出现是为了遮盖父类的Base.doSomething方法，也就是i期望父类的静态方法不要做破坏子类的业务行为，而覆写是将父类的的行为增强或减弱，延续父类的职责。

### 构造函数尽量简化
子类和父类的实例化顺序：首先初始化父类(注意这里是初始化，不是生成父类对象)，也就是初始化父类的变量，调用父类的构造函数，然后才会初始化子类的变量，调用子类的构造函数，最后生成一个实例对象。

在父类构造函数调用子类方法时要考虑子类变量并未赋值而导致的错误。

### 使用构造代码块精简程序
* 普通代码块：就是在方法后面使用"{}"括起来的代码片段，它不能单独运行，必须通过方法名调用执行；
* 静态代码块：在类中使用static修饰，并用"{}"括起来的代码片段，用于静态变量初始化或对象创建前的环境初始化。
* 同步代码块：使用synchronized关键字修饰，并使用"{}"括起来的代码片段，它表示同一时间只能有一个线程进入到该方法块中，是一种多线程保护机制。
* 构造代码块：在类中没有任何前缀和后缀,并使用"{}"括起来的代码片段；
在通过new关键字生成一个实例时会先执行构造代码块，然后再执行其他代码，构造代码块会在每个构造函数内首先执行

遇到this关键字(也就是构造函数调用自身的其它构造函数时)，则不插入构造代码块

作用：统计某个类的实例化次数
```java
// 对象计数器
private static int numOfObjects = 0;

{
    // 构造代码块，计算产生的对象数量
    numOfObjects++;
}
```

### 使用静态内部类提封装性
Java中的嵌套类(Nested Class)分为两种：静态内部类(也叫静态嵌套类，Static Nested Class)和内部类(Inner Class)。

是内部类，并且是静态(static修饰)的即为静态内部类，只有在是静态内部类的情况下才能把static修饰符放在类前，其它任何时候static都是不能修饰类的。

两个优点：加强了类的封装和提高了代码的可读性
```java
public class Person {
    // 姓名
    private String name;
    // 家庭
    private Home home;

    public Person(String _name) {
        name = _name;
    }

    /* home、name的setter和getter方法略 */

    public static class Home {
        // 家庭地址
        private String address;
        // 家庭电话
        private String tel;

        public Home(String _address, String _tel) {
            address = _address;
            tel = _tel;
        }
        /* address、tel的setter和getter方法略 */
    }
}
```
* 提高封装性：从代码的位置上来讲，静态内部类放置在外部类内，其代码层意义就是，静态内部类是外部类的子行为或子属性，两者之间保持着一定的关系，比如在我们的例子中，看到Home类就知道它是Person的home信息。
* 提高代码的可读性：相关联的代码放在一起，可读性肯定提高了。
* 形似内部，神似外部：静态内部类虽然存在于外部类内，而且编译后的类文件也包含外部类(格式是：外部类+$+内部类)，但是它可以脱离外部类存在，也就说我们仍然可以通过new Home()声明一个home对象，只是需要导入"Person.Home"而已。

静态类内部类和普通内部类有什么区别呢？
1. 静态内部类不持有外部类的引用：在普通内部类中，我们可以直接访问外部类的属性、方法，即使是private类型也可以访问，这是因为内部类持有一个外部类的引用，可以自由访问。而静态内部类，则只可以访问外部类的静态方法和静态属性(如果是private权限也能访问，这是由其代码位置决定的)，其它的则不能访问。
2. 静态内部类不依赖外部类：普通内部类与外部类之间是相互依赖关系，内部类实例不能脱离外部类实例，也就是说它们会同生共死，一起声明，一起被垃圾回收，而静态内部类是可以独立存在的，即使外部类消亡了，静态内部类也是可以存在的。
3. 普通内部类不能声明static的方法和变量：普通内部类不能声明static的方法和变量，注意这里说的是变量，常量(也就是final static 修饰的属性)还是可以的，而静态内部类形似外部类，没有任何限制。

### 使用匿名类的构造函数
```java
public static void main(String[] args) {
    List list1=new ArrayList();
    List list2=new ArrayList(){};
    List list3=new ArrayList(){{}};
    System.out.println(list1.getClass() == list2.getClass());
    System.out.println(list2.getClass() == list3.getClass());
    System.out.println(list1.getClass() == list3.getClass());
}
```
list2 = new ArrayList(){}：list2代表的是一个匿名类的声明和赋值，它定义了一个继承于ArrayList的匿名类，只是没有任何覆写的方法而已

list3 = new ArrayList(){{}}：带了两对{}，这也是一个匿名类的定义，就是多了一个初始化块而已
