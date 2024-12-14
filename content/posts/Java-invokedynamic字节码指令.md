+++
date = '2024-12-13T19:14:55+08:00'
draft = false
title = 'Java-invokedynamic字节码指令'
+++

我只知道JVM中有5个invoke字节码指令，但是对常用的lambda表达式相关的`invokedynamic`知之甚少，所以在初步了解之后，简单整理了一下这写内容。


## 前言介绍

Java语言规范对这5个指令的说明如下，前四个比较简单直接：

- `invokestatic`: 调用类方法；
- `invokevirtual`: 调用实例方法，多态的体现；
- `invokespecial`: 也是调用实例方法，但是和`invokevirtual`的区别是：该指令用于直接调用实例的初始化方法（构造函数）以及当前类和其父类的方法，这种调用方式不会考虑对象的实际类型，而是根据方法的声明类型来调用；而`invokevirtual`则会在运行时动态的根据实例的类型去寻找实际的方法来调用；
- `invokeinterface`: 调用接口方法；
- `invokedynamic`: 调用动态计算的调用点。

> [Java语言规范](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-6.html#jvms-6.5.invokedynamic)


## 简单的lambda表达式

### Java源码

简单写一个lambda表达式：

```java
import java.util.function.Function;

public class Main {
  public static void main(String[] args) {
    Function<Integer, Boolean> comparator = number -> number > 10;
    System.out.println(comparator.apply(11));
    System.out.println(comparator.apply(10));
  }
}

```

### 字节码

编译这个类，然后反编译查看字节码：

> javap -c -v -p Main.class

忽略常量池、接口表、父类表等信息，方法表（忽略隐式的无参构造方法，同时删除了操作数栈和本地变量表等影响分析的内容）是这样的：

```
{
  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=2, args_size=1
         0: invokedynamic #7,  0              // InvokeDynamic #0:apply:()Ljava/util/function/Function;
         5: astore_1
         6: getstatic     #11                 // Field java/lang/System.out:Ljava/io/PrintStream;
         9: aload_1
        10: bipush        11
        12: invokestatic  #17                 // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
        15: invokeinterface #23,  2           // InterfaceMethod java/util/function/Function.apply:(Ljava/lang/Object;)Ljava/lang/Object;
        20: invokevirtual #28                 // Method java/io/PrintStream.println:(Ljava/lang/Object;)V
        23: getstatic     #11                 // Field java/lang/System.out:Ljava/io/PrintStream;
        26: aload_1
        27: bipush        10
        29: invokestatic  #17                 // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
        32: invokeinterface #23,  2           // InterfaceMethod java/util/function/Function.apply:(Ljava/lang/Object;)Ljava/lang/Object;
        37: invokevirtual #28                 // Method java/io/PrintStream.println:(Ljava/lang/Object;)V
        40: return

  private static java.lang.Boolean lambda$main$0(java.lang.Integer);
    descriptor: (Ljava/lang/Integer;)Ljava/lang/Boolean;
    flags: (0x100a) ACC_PRIVATE, ACC_STATIC, ACC_SYNTHETIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: invokevirtual #34                 // Method java/lang/Integer.intValue:()I
         4: bipush        10
         6: if_icmple     13
         9: iconst_1
        10: goto          14
        13: iconst_0
        14: invokestatic  #38                 // Method java/lang/Boolean.valueOf:(Z)Ljava/lang/Boolean;
        17: areturn
}
SourceFile: "Main.java"
BootstrapMethods:
  0: #72 REF_invokeStatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
    Method arguments:
      #67 (Ljava/lang/Object;)Ljava/lang/Object;
      #68 REF_invokeStatic com/samakita/payment/transaction/Main.lambda$main$0:(Ljava/lang/Integer;)Ljava/lang/Boolean;
      #71 (Ljava/lang/Integer;)Ljava/lang/Boolean;
```

### 原理解释

可以看到lambda语句的代码被成了一个私有静态方法`private static Boolean lambda$main$0(Integer)`，并且声明lambda语句的代码对应的字节码为`InvokeDynamic #0:apply:()Ljava/util/function/Function;`。

后者的`#0`其实指的是`BootstrapMethods`中的第0个元素，也就是说当JVM解析执行到这一行指令的时候，会根据`BootstrapMethods[0]`的内容去生成一个Function接口的实现子类。

然后看BootstrapMethods表的第0位的内容，有两项：

- 第一项指向的是一个方法：`java.lang.invoke.LambdaMetafactory#metafactory`；
- 第二项是这个方法的参数：
  
  - [1] 第一个参数表示这个lambda表达式实际实现的方法的参数签名【（参数类型）返回值类型】，由于Java的泛型设计为**类型擦除**，所以Function的apply方法的方法签名为：`(Object)Object`;
  - [2] 第二个参数表示生成出来的Function接口的子类的apply方法的实际实现，也就是指向了生成出来的静态方法`lambda$main$0`;
  - [3] 由于在编译时，这个lambda表达式的泛型类型是确定的，所以第三项表示生成出来的静态方法的参数签名：`(Integer)Boolean`;

### ShowMeTheCode

根据**原理解释**一节，我们可以还原出JVM在解释执行到invokedynamic这个字节码的时候做的事情，代码大概类似这样：

```java
import java.lang.invoke.CallSite;
import java.lang.invoke.LambdaMetafactory;
import java.lang.invoke.MethodHandle;
import java.lang.invoke.MethodHandles;
import java.lang.invoke.MethodType;
import java.util.function.Function;

public class MainExplain {

  /** lambda表达式对应的私有静态方法 */
  private static Boolean lambda$main$0(Integer number) {
    return number > 10;
  }

  public static void main(String... args) throws Throwable {
    MethodHandles.Lookup caller = MethodHandles.lookup();
    // 生成出来的类的类型
    MethodType factoryMethodType = MethodType.methodType(Function.class);
    // [1] lambda表达式实现的方法的参数签名，第一个为返回值类型，后边为参数类型
    MethodType interfaceMethodType = MethodType.methodType(Object.class, Object.class);
    // [3] lambda表达式生成出来的静态方法的参数签名
    MethodType dynamicMethodType = MethodType.methodType(Boolean.class, Integer.class);
    // [2] 从当前类中查找静态方法
    MethodHandle implementation =
        caller.findStatic(caller.lookupClass(), "lambda$main$0", dynamicMethodType);
    // 动态生成实现类
    CallSite callSite =
        LambdaMetafactory.metafactory(
            caller,
            "apply",
            factoryMethodType,
            interfaceMethodType,
            implementation,
            dynamicMethodType);
    // 获取实例
    Function<Integer, Boolean> invoke =
        (Function<Integer, Boolean>) callSite.dynamicInvoker().invoke();
    // 调用方法
    System.out.println(invoke.apply(11)); // true
    System.out.println(invoke.apply(10)); // false
  }
}
```

`metafactory`方法先创建了一个`InnerClassLambdaMetafactory`对象，然后通过这个对象来构建`callSite`，本质上是使用asm技术动态生成了一个类，在这种场景下这个类的构成比较明确，就不反编译贴出来了。调用链路如下：

```java
java.lang.invoke.AbstractValidatingLambdaMetafactory#buildCallSite
  --> java.lang.invoke.InnerClassLambdaMetafactory#spinInnerClass
      --> java.lang.invoke.InnerClassLambdaMetafactory#generateInnerClass
```

## 引用外部变量的lambda表达式

### Java源码

写一个引用外部变量的lambda表达式如下：

```java
import java.util.List;
import java.util.function.Function;
import java.util.stream.IntStream;

public class Main {
  public static void main(String[] args) {
    String content = getContent();
    Function<Integer, List<String>> repeat =
        count -> IntStream.range(0, count).mapToObj(i -> content).toList();
    System.out.println(repeat.apply(10));
  }

  private static String getContent() {
    return "CONTENT";
  }
}
```

### 字节码

然后查看字节码，同样只粘贴关键部分：

```
{
  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=3, args_size=1
         0: invokestatic  #7                  // Method getContent:()Ljava/lang/String;
         3: astore_1
         4: aload_1
         5: invokedynamic #13,  0             // InvokeDynamic #0:apply:(Ljava/lang/String;)Ljava/util/function/Function;
        10: astore_2
        11: getstatic     #17                 // Field java/lang/System.out:Ljava/io/PrintStream;
        14: aload_2
        15: bipush        10
        17: invokestatic  #23                 // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
        20: invokeinterface #29,  2           // InterfaceMethod java/util/function/Function.apply:(Ljava/lang/Object;)Ljava/lang/Object;
        25: invokevirtual #34                 // Method java/io/PrintStream.println:(Ljava/lang/Object;)V
        28: return

  private static java.util.List lambda$main$1(java.lang.String, java.lang.Integer);
    descriptor: (Ljava/lang/String;Ljava/lang/Integer;)Ljava/util/List;
    flags: (0x100a) ACC_PRIVATE, ACC_STATIC, ACC_SYNTHETIC
    Code:
      stack=2, locals=2, args_size=2
         0: iconst_0
         1: aload_1
         2: invokevirtual #42                 // Method java/lang/Integer.intValue:()I
         5: invokestatic  #46                 // InterfaceMethod java/util/stream/IntStream.range:(II)Ljava/util/stream/IntStream;
         8: aload_0
         9: invokedynamic #52,  0             // InvokeDynamic #1:apply:(Ljava/lang/String;)Ljava/util/function/IntFunction;
        14: invokeinterface #55,  2           // InterfaceMethod java/util/stream/IntStream.mapToObj:(Ljava/util/function/IntFunction;)Ljava/util/stream/Stream;
        19: invokeinterface #59,  1           // InterfaceMethod java/util/stream/Stream.toList:()Ljava/util/List;
        24: areturn
}
SourceFile: "Main.java"
BootstrapMethods:
  0: #105 REF_invokeStatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
    Method arguments:
      #92 (Ljava/lang/Object;)Ljava/lang/Object;
      #93 REF_invokeStatic com/samakita/payment/transaction/Main.lambda$main$1:(Ljava/lang/String;Ljava/lang/Integer;)Ljava/util/List;
      #96 (Ljava/lang/Integer;)Ljava/util/List;
```

### 比较差异

可以发现，这里生成的私有静态方法`lambda$main$1`的方法签名【List lambda$main$1(String, Integer)】跟实际的`FunctionalInterface`接口的方法签名【R apply(T t)】的数量不一致的，也就是说lambda表达式引用的外部变量（final的），会在生成出来的静态私有方法的参数上添加一个对应类型的参数。

现在有几个问题：

1. JDK动态生成的类和之前简单的lambda场景下有何区别；
2. content这个String类型的变量是如何在调用`apply`方法的时候传给`lambda$main$1`的呢；

### 结论

首先先将字节码还原到`metafactory`调用：

```java
import java.lang.invoke.CallSite;
import java.lang.invoke.LambdaMetafactory;
import java.lang.invoke.MethodHandle;
import java.lang.invoke.MethodHandles;
import java.lang.invoke.MethodType;
import java.util.List;
import java.util.function.Function;
import java.util.stream.IntStream;

public class MainExplain {

  /** lambda表达式对应的私有静态方法 */
  private static List<String> lambda$lambda$1(String content, Integer count) {
    return IntStream.range(0, count).mapToObj(i -> content).toList();
  }

  public static void main(String... args) throws Throwable {
    MethodHandles.Lookup caller = MethodHandles.lookup();
    // 生成出来的类的类型
    // [1] 这里和简单场景下有一点不同，需要一个额外的String类型的参数
    MethodType factoryMethodType = MethodType.methodType(Function.class, String.class);
    // lambda表达式实现的方法的参数签名，第一个为返回值类型，后边为参数类型
    MethodType interfaceMethodType = MethodType.methodType(Object.class, Object.class);
    // lambda表达式生成出来的静态方法的参数签名
    MethodType implementionMethodType =
        MethodType.methodType(List.class, String.class, Integer.class);
    // 从当前类中查找静态方法
    MethodHandle implementation =
        caller.findStatic(caller.lookupClass(), "lambda$lambda$1", implementionMethodType);
    MethodType dynamicMethodType = MethodType.methodType(List.class, Integer.class);
    // 动态生成实现类
    CallSite callSite =
        LambdaMetafactory.metafactory(
            caller,
            "apply",
            factoryMethodType,
            interfaceMethodType,
            implementation,
            dynamicMethodType);
    String content = Main.getContent();
    // [2] 获取实例，需要将引用的外部变量传递到invoke方法中
    Function<Integer, List<String>> invoke =
        (Function<Integer, List<String>>) callSite.dynamicInvoker().invoke(content);
    // 调用方法
    System.out.println(invoke.apply(10));
  }
}
```

可以看到有两个不一样的地方：

- [1] factoryMethodType的参数原先是单一个`Function.class`，现在是`Function.class, String.class`
- [2] 获取Function实例的时候，将content这个变量传到了`invoke`方法中

这样可以回答第一个问题。将生成出来的类反编译出来，看看第一点这个变化构造出来的类是什么结构，并且也可以回答第二个问题：

```java
import java.util.function.Function;

final class MainExplain$$Lambda implements Function {
    private final String arg$1;

    private MainExplain$$Lambda(String var1) {
        this.arg$1 = var1;
    }

    public Object apply(Object var1) {
        return MainExplain.lambda$lambda$1(this.arg$1, (Integer)var1);
    }
}
```

可以看到invoke方法可以理解成调用这个动态类的构造函数，content会被存放到final关键词修饰的成员变量中。然后apply方法执行时，会自动添加自身的final成员变量作为方法的参数。