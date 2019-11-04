# 第3章 不假思索：我精通 Java

> 大家都知道Java 是 Android 开发者必备的技术，也是后续高级话题的切入点。在这一点上，我们没有丢分的理由。

**你对 Java 究竟有多少了解？**

#3-1 Java 的 char 是两个字节，是怎么存 UTF-8 的字符的？

##面试官视角：这道题想考察什么？

* 是否熟悉 Java char 和字符串（初级）
* 是否了解字符的映射和存储细节（中级）
* 是否能触类旁通，横向对比其他语言（高级）

## 题目剖析

* Java的 **char** 是两个字节，如何存 **UTF-8**字符？

  * 剖析题目技巧：找名词，解释名词，关联名词

  * 两个名词：char、UTF-8

  * 只需要把char是什么，UTF-8是什么讲清楚了，再把他们俩之间的关系讲出来，那么这个题目基本上就回答对了一半了。

  * 开始剖析

    * char u8Test = '庆';    //Java char占两个字节

    * UTF-8 Bytes

       | 字符 | UTF-8 字节 |
       | ---- | ---------- |
       | 庆   | e5ba86     |
       | 祝   | e7a59d     |
       | 改   | e694b9     |
       | 革   | e99da9     |
       | 开   | e5bc80     |
       | 放   | e694be     |
       | 4    | 34         |
       | 0    | 30         |
       | 周   | e591a8     |
       | 年   | e5b9b4     |
  
  * 题目陷阱
  
    * 面试题经常故意设陷阱！
    * 但你不能说 “题出错了！”
    * 题目只是给定范围，是话题作文不是命题作文！
  
    
  
  1. Char 是什么？占几个字节？
  2. UTF-8 是什么？占几个字节？
  3. 和Unicode什么关系？

## 题目剖析

* Java char 不存 UTF-8 的字节，而是 UTF-16
* Unicode 通用字符集占两个字节，例如 “中”
* Unicode 扩展字符集需要用一对 char 来表示，例如 “😀”
* Unicode 是字符集，不是编码，作用类似于 ASCII 码
* Java String 的length 不是字符数，而是char的个数

##技巧点拨

* 抓住细节，有技巧的回避知识盲区
* 把我节奏，不要等面试官追问
* 主动深入，让面试官了解你的知识体系
* 触类旁通，让面试官眼前一亮

# 3-2 Java String 可以有多长？

* 是否对字符串编解码有深入了解（中级）
* 是否对字符串在内存当中的存储形式有深入了解（高级）
* 是否对Java虚拟机字节码有足够的了解（高级）
* 是否对Java虚拟机指令有一定认识（高级）

## 题目剖析

###Java <font color=red>**String**</font> 可以有多 <font color=red>**长**</font> ？

* 字符串有多长是指字符数还是字节数？

* 字符串有

  ------

  几种存在形式？

* 字符串的不同形式受到何种限制？

### Java String ....

* 栈

  ```java
  String longString = "aaa...aaa"; // n = ?
  ```

* 堆

  ```java
  byte[] bytes = loadFromFile(new File("superLongText.txt"))
  String superLongString = new String(bytes)
  ```

### Java String <font color=red>**栈**</font>

源文件：*.java			String longString = "aaa...aaa"; // n <= 65535

------

字节码：*.class			CONSTANT_Utf8_info {  // 字符串在字节码中是utf8的结构体存储

​											u1 tag;  

​											u2 length;     // u2 是一个两个字节的值，两个字节最大65535，取值区间0~65535

​											u1 bytes[length];    最多65535个字节

​										}

------

虚拟机：常量池			Java虚拟机内存 -包含-> 方法区 -包含-> 常量池



String longString = "aaa...aaa"; // n = 65535

上面这段代码会编译失败：<font color=red>**Error: java:constans string too long**</font>

当n = 65534时编译通过，为什么？

**原因（Javac 的源码，Gen.java）：**

```java
private void checkStringConstant(DiagnosticPosition pos, Object ConstValue) {
    if (nerrs != 0 || //only complain about a long string once
       	constValue == null ||
       	!(constValue instanceof String) ||
        ((String)constValue).length() < Pool.MAX_STRING_LENGTH) //常量值为65535，这里是小于则校验通过
       return;
    log.error(pos, "limit.string");
    nerrs++;
}
```



源文件：*.java			String longString = "烫烫烫...烫烫烫"; // n = 65535 / 3

​										汉字“烫”占3个字节

------

字节码：*.class			CONSTANT_Utf8_info {  // 字符串在字节码中是utf8的结构体存储

​											u1 tag;  

​											u2 length;     // u2 是一个两个字节的值，两个字节最大65535，取值区间0~65535

​											u1 bytes[length];    最多65535个字节

​										}

String longString = "烫烫烫...烫烫烫"; 

当n = 65535 / 3时，编译通过，这又是为什么？

**原因（Javac 的源码，writePool.java）：**

```java
...
	} else if (value instanceof Name) {
        poolbuf.appendByte(CONSTANT_Utf8);
        byte[] bs = ((Name)value).toUtf();
        poolbuf.appendChar(bs.length);
        poolbuf.appendBytes(bs, 0, bs.length);
        if (bs.length > Pool.MAX_STRING_LENGTH) {//常量值为65535，这里是大于则抛异常
            throw new StringOverflow(value.toString());
        }
    }
...
```

**总结**

* 受字节码限制，字符串最终的 MUTF-8 字节数不超过 65535
* Latin 字符，受 Javac 代码限制，最多 655534 个
* 非Latin 字符最终对应字节个数差异较大，最多字节个数是65535
* 如果运行时方法区设置较小，也会受到方法区大小的限制



### Java String <font color=red>**堆**</font>

```java
byte[] bytes = loadFromFile(new File("superLongText.txt"));

String superLongString =  new String(bytes);
private final char value[];
```

<font color=red>虚拟机指令</font>：newarray [int]  数组理论最大个数为Integer.MAX_VALUE

```java
/**	
* The maximum size of array to allocate.
* Some VMs reserve some header words in an array.
* Attempts to allocate larger arrays may result in
* OutOfMemoryError: Requested array size excedds VM limit
*/
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

**总结**

* 受虚拟机指令限制，字符数理论上限为Integer.MAX_VALUE
* 受虚拟机实现限制，实际上限可能会小于Integer.MAX_VALUE
* 如果堆内存较小，也会受到堆内存的限制



### 本节回顾

推荐两本书：《Java虚拟机规范》、《Java语言规范》；这样在Java语言层面就没有人能用问题难倒你了。

* Java String 字面量形式
  * 字节码 CONSTANT_Utf8_info 的限制
  * Javac 源码逻辑的限制
  * 方法区大小的限制
* Java String 运行时创建在堆上的形式
  * Java 虚拟机指令 newarray 的限制
  * Java 虚拟机堆内存大小的限制

### 技巧点拨

* 思路很重要！
  * 这种类型的题目最终的结果往往不重要
  * 拿到问题，知道如何分析，知道从哪儿分析是关键
* 切不可眼高手低！
  * 简单的问题背后暗藏玄机
  * 尽一切可能将题目引向自己擅长的领域



#3-3 Java 的匿名内部类有哪些限制？

## 面试官视角：这道题想考察什么？

* 考察匿名内部类的概念和用法（初级）
* 考察语言规范以及语言的横向对比等（中级）
* 作为考察内存泄漏的切入点（高级）

###③ 匿名内部类的构造方法

* 编译器生成
* 参数列表包括
  * 外部对象（定义在非静态域内）
  * 父类的外部对象（父类非静态）
  * 父类的构造方法参数（父类有构造方法且参数列表不为空）
  * 外部捕获的变量（方法体内有引用外部 final 变量）

**庖丁解牛**：

* 自己的外部类实例、非静态父类的外部类实例

```java
public class OuterClass {
    public abstract class InnerClass {
        abstract void test();
    }
}
```

```java
public class Client {
    public void run() {
        InnerClass innerClass =
            new OuterClass().new InnerClass() {
            ...
        }
    }
}
```

```java
public class Client$1 {
    // Client client: 自己的外部实例；
    // OuterClass outerClass: 非静态父类的外部类实例
    public Client$1(Client client, OuterClass outerClass) {
        ...
    }
}
```



* 外部捕获的变量（方法体内有引用外部 final 变量）

```java
public class OuterClass {
    public interface InnerClass {
        void test();
    }
}
```

```java
public class Client {
    public static void run() {
        final Object object = new Object();
        InnerClass innerClass = 
            new OuterClass.InnerClass() {
            @Override
            void test() {
                System.out.println(object.toString());
            }
        }
    }
}
```

解析：

```java
public class Client$1 {
    public Client$1(Object object) {
        ...
    }
}
```

**上述代码中Client$1则为<font color=red>编译器自动生成</font>的匿名内部类名字，Object object ：<font color=red>捕获外部变量</font>**

###④ Lambda转换（SAM类型）

错误使用方式：

CallBack callBack = () -> {

...

}

对应

```java
abstract class CallBack {
    abstract void onSuccess();
}
```

正确使用方式：

Call callBack = () -> {

...

}

对应

```java
interface CallBack {
    void onSuccess();
}
```

错误使用方式：

Call callBack = () -> {

...

}

对应

```java
interface CallBack {
    void onSuccess();
    void onError();
}
```

## 本节回顾

* 没有人类认知意义上的名字
* 只能继承一个父类或实现一个接口
* 父类是非静态的类型，则需父类外部实例来初始化
* 如果定义在非静态作用域内，会引用外部类实例
* 只能捕获外部作用域内的 final 变量
* 创建时只有单一方法的接口可以用 Lambda转换

### 技巧点拨

* 关注语言版本的变化
  * 体现对技术的热情
  * 体现好学的品质
  * 显得专业

#3-4 怎样理解 Java 的方法分派？

 ## 面试官视角：这道题想考察什么？

* 多态、虚方法表的认识（初级）
* 对编译和运行时的理解和认识（中级）
* 对 Java 语言规范和运行机制的深入认识（高级）
* 横向对比各类语言的能力（高级）
  * Groovy，Gradle DSL 5.0 以前唯一正式语言
  * C++，Native 程序开发必备

## 题目剖析

* 怎么理解 Java 的<font color=red> **方法分派** </font>？
  * 就是确定调用**谁的**、**哪个**方法？
  * 针对方法重载的情况进行分析
  * 针对方法覆写的情况进行分析

## 方法分派 示例

```java
class SuperClass {
    public String getName() {
        return "Super";
    }
}

class SubClass extends SuperClass {
    public String getName() {
        return "Sub";
    }
}
```

```java
public class Question4 {
    public static void main(String... args) {
        SuperClass superClass = new SubClass();
        printHello(superClass);
    }
    
    public static void printHello(SuperClass superClass) {
        System.out.println("Hello " + superClass.getName());
    }
    
    public static void printHello(SubClass subClass) {
        System.out.println("Hello " + subClass.getName());
    }
}
```

<font color=red> **问题一：程序输出什么？**</font>

---> Hello Sub

**取决于运行时的**<font color=red>**实际类型**</font>

<font color=red> **问题二：程序如何执行？**</font>

**取决于编译时的**<font color=red>**声明类型**</font>

```java
SuperClass superClass = new SubClass();
printHello(superClass);
```

​			↓

```java
ALDAD 1
INVOKESTATIC com/bennyhuo/iiv/ch1/Question4, printHello (Lcom/bennyhuo/iiv/ch1/SuperClass;)V
```

## Java 方法分派

* 静态分派 - 方法重载分派
  * 编译期确定
  * 依据调用者的声明类型和方法参数类型
* 动态分派 - 方法覆写分派
  * 运行时确定
  * 依据调用者的实际类型分派

## 触类旁通（1）

**Groovy**

```java
class SuperClass {
    public String getName() {
        return "Super";
    }
}

class SubClass extends SuperClass {
    public String getName() {
        return "Sub";
    }
}
```

```java
public class Question4 {
    public static void main(String... args) {
        SuperClass superClass = new SubClass();
        printHello(superClass);
    }
    
    public static void printHello(SuperClass superClass) {
        System.out.println("Hello " + superClass.getName());
    }
    
    public static void printHello(SubClass subClass) {
        System.out.println("Hello " + subClass.getName());
    }
}
```

程序输出依然是 Hello Sub

程序如何运行？<font color=red>**取决于运行时的实际类型**</font>

printHello(superClass)编译完成后生成的字节码如下，printHello(superClass)编译结果就是“CallSite.callStatic“方法，这个方法是干什么的？

运行时根据反射拿到实际类型，然后再去决定调用哪一个printHello方法

```
LINENUMBER 6 L3
ALOAD 1
LDC 1
AALOAD
LDC Lcom/bennyhuo/iiv/ch1/Question4;class
ALOAD 2
INVOKEINTERFACE org/codehaus/groovy/runtime/callsite/CallSite.callStatic (Ljava/lang/Class;Ljava/lang/Object;)Ljava/lang/Object;
```

##触类旁通（2）

**C++**

```c++
class SuperClass {
    public:
    	string getName() { // 非虚方法
            return "Super";
        }
};

class SubClass: public SuperClass {
    public:
   		string getName() {
            return "Sub";
        }
}
```

```c++
int main() {
    SuperClass superClass = SubClass();// 栈内存
    cout << superClass.getName() << endl;
    
    SuperClass *superClassPtr = new SubClass();// 堆内存
    out << superClassPtr->getName() << endl;
    
   	delete(superClassPtr);
}
```

输出结果：

→ Super

→ Super

原因：上述SuperClass类的getName不是虚方法

##触类旁通（2-1）

**C++**

```c++
class SuperClass {
    public:
    	virtual string getName() { // 虚方法
            return "Super";
        }
};

class SubClass: public SuperClass {
    public:
   		string getName() {
            return "Sub";
        }
}
```

```c++
int main() {
    SuperClass superClass = SubClass();// 栈内存
    cout << superClass.getName() << endl;
    
    SuperClass *superClassPtr = new SubClass();// 堆内存
    out << superClassPtr->getName() << endl;
    
   	delete(superClassPtr);
}
```

输出结果：

→ Super（原因：对象裁剪。Sub -> Super，因为Sub的所需内存比Super大，然后赋值给Super时，内存会被剪切到跟Super一样大，此时Sub子类的getName方法已经被剪切掉了，只能调用到父类Super的getName方法）

→ Sub

##触类旁通（2-2）

**C++**

```c++
void printHello(SuperClass superClass) {
    cout << superClass.getName() << endl;
}
void printHello(SuperClass superClass) {
    cout << superClass.getName() << endl;
}
void printHello(SubClass subClass) {
    cout << subClass.getName() << endl;
}
void printHello(SuperClass* superClass) {
    cout << superClass -> getName() << endl;
}
void printHello(SubClass* subClass) {
    cout << subClass -> getName() << endl;
}
```

```c++
SuperClass superClass = SubClass();
SuperClass*subperClassPtr = new SubClass();
printHello(superClass);
printHello(*superClassPtr);
printHello(&superClass);
printHello(superClassPtr);
```

输出结果：

→ Super

→ Super

→ Super

→ Sub （原因：调用了虚方法）

还有一种参数引用类型：

```c++
void printHello(SuperClass& superClass) {
    cout << superClass.getName() << endl;
}
```

```c++
SuperClass superClass = SubClass();
SuperClass*subperClassPtr = new SubClass();
printHello(*superClassPtr);
```

输出结果：Sub

## 本节回顾

* 分析 Java 方法重载时的分派行为
* 分析 Java 方法覆写时的分派行为
* 横向对比 Groovy 与 C++ 的分派行为

## 技巧点拨

* 横向对比
  * 体现扎实的语言基本功
  * 体现对于编程语言特性的专研精神
  * 显得专业

# 3-5 Java 泛型的实现机制是怎样的？

## 面试官视角：这道题想考察什么？

* 对 Java 泛型使用是否仅停留在集合框架的使用（初级）
* 对 Java 泛型的实现机制的认知和理解（中级）
* 是否有足够的项目开发实战和 “踩坑” 经验（中级）
* 对泛型（或模板）编程是否有深入的对比研究（高级）
* 对常见的框架原理是否有过深入剖析（高级）

## 题目剖析

* 题目区分度非常大
* 回答需要提及以下几点才能显得有亮点
  * 类型擦除从编译角度的细节
  * 类型擦除对运行时的影响
  * 对比类型不擦除的语言
  * 为什么 Java 选择类型擦除
* 可从类型擦除的优劣来着手分析回答

## 类型擦除有哪些好处？

<font color=red>**运行时内存负担小**</font>

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/破解Android高级面试/img/030501.png)

 <font color=red>**兼容性好**</font>

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/破解Android高级面试/img/030502.png)

<font color=red>**基本类型无法作为泛型实参**</font>

google为Android专门提供了SparseArray，避免了装箱拆箱的操作

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/破解Android高级面试/img/030503.png)

<font color=red>**泛型类型无法用作方法重载** </font>

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/破解Android高级面试/img/030504.jpg)

 编译时类型被擦除了

<font color=red>**泛型类型无法当做真实类型使用** </font>

类型在编译时会被擦除，T编译为Object了，new Object()显然不合适

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/破解Android高级面试/img/030505.jpg)

**Java**

```java
static <T> void genericMethod(T t) {
    T newInstance = new T(); // Error: 无法创建类型
    T[] array = new T[0]; // Error: 无法创建数组
    Class c = T.class; // Error: 无法获取类型
    List<T> list = new ArrayList<T>(); // Ok: 用于其他泛型类型
    if (list instanceof List<String>) {}; // Error: 非法的类型判断
}
```

**C#**

```c#
static void GenericMethod<T>(T t) where T:new(){
    T newStance = new T(); // OK: T受new()约束
    T[] array = new T[0]; // OK: 可以创建数组
    Type type = typeof(T); // OK: typeof 获取类型
    List<T> list = new List<T>(); // OK: 用于其他泛型类型
    if (list is List<String>){}; // OK: 泛型类是真实类型
}
```

<font color=red>**知识迁移：Gson.fromJson 为什么需要传入Class**</font>

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/破解Android高级面试/img/030506.jpg)

原因：类型擦除

<font color=red>**静态方法无法引用类泛型参数**</font>

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/破解Android高级面试/img/030507.jpg)

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/破解Android高级面试/img/030508.jpg)

 类泛型参数只有在类实例化时才知道，而静态方法不需要类的实例，所以无法引用类型T； 

<font color=red>**类型强转的运行时开销**</font>

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/破解Android高级面试/img/030509.jpg)

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/破解Android高级面试/img/030510.jpg)

类型擦除

<font color=red>**附加的签名信息**</font>

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/破解Android高级面试/img/030511.jpg)

```java
class SuperClass<T>{}
class SubClass extends SuperClass<String> {
    public List<Map<String, Integer>> getValue(){ return null; }
}
```

```java
ParameterizedType methodType = 
    (ParameterizedType) SubClass.class.getGenericReturnType();
for (Type actualTypeArgument : superType.getActualTypeArguments()) {
    System.out.println(actualTypeArgument);
}
```

—> class.java.lang.String

```java
ParameterizedType methodType = 
    (ParameterizedType) SubClass.class.getMethod("getValue").getGenericReturnType();
for (Type type : methodType.getActualTypeArguments()) {
    System.out.println(type);
}
```

----> java.util.Map<java.lang.String, java.lang.Integer>

<font color=red>**混淆时要保留签名信息**</font>

Proguard

-Keepattributes Signature

<font color=red>**迁移：使用泛型签名的两个实例** </font>

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/破解Android高级面试/img/030512.jpg)

Gson**

```java
Type collectionType = new TypeToken<Collection<Integer>>(){}.getType();
Collection<Integer> ints = gson.fromJson(json, collectionType)
```

**Retrofit**

```java
interface GitHubServiceApi {
    @GET("users/{login}")
    Call<User> getUserCallback(@path("login")String login);
}
```

<font color=red>**迁移：Kotlin 反射的实现原理**</font>

**Kotlin**

```java
class HelloWorld
```

```java
public final class com/bennyhuo/iiv/ch1/HelloWorld {
    // access flags 0x1
    public <init>()V
    ...
    @LKotlin/Metadata;(mv={1,1,13}}, bv={1,0,3}, k=1, d1={"\u000\u000c\n\u0002...."}，
d2 = {"Lcom/bennyhuo/iiv/ch1/HelloWorld;","","()v","Chapter1_JavaBasic_main"})
    // compiled from; HelloWorld.kt
}
```

## 本节回顾

* Java 泛型采用类型擦除实现
* 类型编译时被擦除为Object，不兼容基本类型
* 类型擦除的实现方案主要考虑后向兼容
* 泛型类型签名信息特定场景下反射可获取

## 技巧点拨

* 结合项目实践
  * 阐述观点给出实际案例，例如Gson、Retrofit
  * 实战中经常需要混淆，需要注意哪些点以及原理

# 3-6 Activity 的 onActivityResult 使用起来非常麻烦，为什么不设计成回调？

## 面试官视角：这道题想考察什么？

* 是否熟悉 **onActivityResult** 的用法（初级）
* 是否**思考**过用回调替代 **onActivityResult**（中级）
* 是否**实践**过用回调替代 **onActivityResult**（中级）
* 是否意识到回调存在的问题（高级）
* 是否能给出匿名内部类对外部引用的解决方案（高级）

## 题目剖析

* Activity 的 onActivityResult 使用起来非常麻烦，为什么不设计成回调？
  * onActivityResult是干什么的，怎么用
  * 回调在这样的场景下适用吗？
  * 如果适用，那为什么不用回调？
  * 如果不适用，给出你的理由

## onActivityResult 为什么麻烦？

① 代码处理逻辑分离，容易出现遗漏和不一致的问题

② 写法不够直观，且结果数据没有类型安全保障

③ 结果种类较多时，onActivityResult就会逐渐臃肿难以维护

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/破解Android高级面试/img/030601.jpg)

## 假设：用回调实现

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/破解Android高级面试/img/030602.jpg)

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/破解Android高级面试/img/030603.jpg)

## 为什么不使用回调

* **onActivityResult** 确实麻烦
* CallBack 确实也可以简化代码编写
* 但 Activity 的销毁和恢复机制不允许匿名内部类出现

## 基于注解处理器和Fragment 的回调实现

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/破解Android高级面试/img/030604.jpg)

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/破解Android高级面试/img/030605.jpg)

## 思考：捕获了 View和 Fragment 以外的变量需要替换吗？

课堂作业



## 本节回顾

* 分析 onActivityResult 在使用上存在的问题
* 分析为什么 onActivityResult 的场景不设计成回调
* 通过替换匿名内部类的外部引用实现回调

## 本节点拨

* 多想为什么
* 多实践自己的想法