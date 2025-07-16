# Java泛型

> ​ 在日常开发中，必不可少的会使用到泛型，这个过程中经常会出现类似“为什么这样会编译报错？”，“为什么这个列表无法添加元素？”的问题，也会出现感叹Java的泛型限制太多了很难用的情况。为了更好的使用泛型，就需要更深的了解它，因此本文主要介绍泛型诞生的前世今生，特性，以及著名PECS（Producer-Extends, Consumer-Super）原则的由来。

## 泛型的诞生

### 为什么需要泛型

​         在没有泛型之前，必须使用Object编写适用于多种类型的代码，想想就令人头疼，并且非常的不安全。同时由于数组的存在，设计者为了让其可以比较通用的进行处理，也让**数组允许协变**，这又为程序添加了一些天然的不安全因素。为了解决这些情况，Java的设计者终于在Java5中引入泛型，然而，正是因为引入泛型的时机较晚，为了兼容先前的代码，设计者也不得不做出一些限制，来让使用者以难受换来一些安全。

1. **类型安全**：编译时进行类型检查，避免运行时`ClassCastException`
2. **消除强制类型转换**：使代码更简洁易读
3. **代码复用**：可以编写更通用的算法和数据结构
4. **更好的API设计**：API可以更清晰地表达其预期的输入和输出类型

简单来说，泛型的引入有以下好处：

- 程序更加易读
- 安全性有所保证

​ 以`ArrayList`举例，在增加泛型类之前，其通用性是用继承来实现的，`ArrayList`类只维护一个Object引用的数组，当我们使用这个工具类时，想要获取指定类型的对象必须经过强转：

```java
package com.demo.genericity;

import java.util.ArrayList;
import java.util.Date;

public class Test {

    public static void main(String[] args) {
        ArrayList list = new ArrayList();
        // 强制类型转换
        String o = (String) list.get(0);
        //十分不安全的行为
        list.add(new Date());
    }
}
```

​ 这种写法在编译类型时不会报错，但一旦使用get获取结果并试图将Date转换为其他类型时，很有可能出现类型转换异常，为了解决这种情况，类型参数应用而生。

### 类型参数

类型参数（Type parameter）使得`ArrayList`以及其他可能用到的集合类能够方便的指示虚拟机其包含元素的类型:

```java
package com.demo.genericity;

import java.util.ArrayList;

public class Test {
    public static void main(String[] args) {
        ArrayList<String> list = new ArrayList();
        list.add("Hello");
    }
}
```

​         这使得代码具有更好的可读性，并且在调用`get()`的时候，无需进行强转，最重要的是，编译器终于可以检查一个插入操作是否符合要求，运行时可能出现的各种类型转换错误得以在编译阶段就被阻止。

![](D:\Users\Desktop\技术分享\Java泛型\微信图片_20250515092523.png)

## 基本用法

### 泛型类

​ 泛型类是有一个或者多个类型变量的类，泛型类中的属性可以**全都不是泛型**，不过一般不会这样做，毕竟类型变量在整个类上定义就是用于指定方法的返回类型以及字段的类型，定义代码如下：

```java
package com.demo.genericity.covariance;

public class Animal<T> {

    private String name;

    private T mouth;

    public T getMouth() {
        return mouth;
    }
}
```

泛型类可以有多个类型变量：

```java
package com.demo.genericity.covariance;

public class Animal<T, U, Z, H, P> {

    private String name;

    private T mouth;

    private U eye;

    private Z foot;

    private H head;

    private P assHole;

    public T getMouth() {
        return mouth;
    }

    public U getEye() {
        return eye;
    }
}
```

```java
public class Box<T> {
    private T t;

    public void set(T t) { this.t = t; }
    public T get() { return t; }
}

// 使用
Box<String> stringBox = new Box<>();
stringBox.set("hello");
String str = stringBox.get();
```

### 泛型接口

```java
public interface Pair<K, V> {
    public K getKey();
    public V getValue();
}

public class OrderedPair<K, V> implements Pair<K, V> {
    private K key;
    private V value;

    public OrderedPair(K key, V value) {
        this.key = key;
        this.value = value;
    }

    public K getKey() { return key; }
    public V getValue() { return value; }
}

// 使用
Pair<String, Integer> p1 = new OrderedPair<>("Even", 8);
```

### 泛型方法

泛型方法可以在普通类中定义，也可以在泛型类中定义，例如：

```java
package com.demo.genericity.covariance;

public class Animal<T, U> {

    private T value;

    public static <T> T get(T... a) {
        return a[a.length - 1];
    }

    public T getFirst() {
        return value;
    }
}
```

### 类型通配符

```java
public static void process(List<? extends Number> list) {
    for (Number elem : list) {
        // 处理Number及其子类的元素
    }
}
```

#### 上界通配符 (<? extends T>)

```java
List<? extends Number> list = new ArrayList<Integer>();
// list.add(new Integer(1)); // 编译错误
Number num = list.get(0); // 可以读取为Number
```

#### 下界通配符 (<? super T>)

```java
List<? super Integer> list = new ArrayList<Number>();
list.add(new Integer(1)); // 可以添加Integer及其子类
// Number num = list.get(0); // 编译错误
Object obj = list.get(0); // 只能读取为Object
```

![](D:\Users\Desktop\技术分享\Java泛型\微信图片_20250515092516.png)

## 类型擦除

![](C:\Users\Administrator\AppData\Roaming\marktext\images\2025-05-15-10-02-14-image.png)

        虚拟机没有泛型类型对象，也就是说，所有对象在虚拟机中都属于普通类，这意味着在程序编译并运行后我们的类型变量会被擦除（erased）并替换为限定类型，擦掉类型参数后的类型就叫做原始类型（raw type），正是因为类型参数被擦除了，所以比较结果会为true:

          Java泛型是通过类型擦除实现的，这意味着在编译时类型参数信息会被移除（擦除），并在必要的地方插入类型转换。

![](C:\Users\Administrator\AppData\Roaming\marktext\images\2025-05-15-10-29-22-image.png)

        前面说过，泛型是在1.5才提出的，因此类型擦除的目的就是为了保证已有的代码和类文件依然合法，也就是向低版本兼容。这样做会带来几个问题：

1. 类型参数不支持基本类型，只支持引用类型，这是因为泛型会被擦除为具体类型，而Object不能存储基本类型的值
   
   ![](C:\Users\Administrator\AppData\Roaming\marktext\images\2025-05-15-10-31-44-image.png)

2. 不能实例化类型参数
   
   不能实例化泛型数组，因为类型擦除会将数组变为Object数组，如果允许实例化，极易造成类型转换异常。

```java
T[] array = new T[]{}; // 编译错误
T obj = new T(); // 编译错误
List<String>[] array = new List<String>[10]; // 编译错误
```

### 强制转换

    在编写泛型方法调用时，如果擦除了返回类型，编译器会插入强制类型转换。例如下面的代码：

![](C:\Users\Administrator\AppData\Roaming\marktext\images\2025-05-15-10-39-27-image.png)

getFirst擦除类型后的返回类型是Object，编译器自动插入转换到Integer的强制类型转换，也就是说，编译器把这个方法调用转换为两条虚拟机指令：

- 对原始方法的调用。

- 将返回的Object类型强制转换为Integer类型。

> 子类重写父类方法时，必须和父类保持相同的方法名称，参数列表和返回类型。那么问题来了，如果按照之前的思路来讲，当泛型父类或接口的类型参数被擦除了，那么子类岂不是不构成重写条件？（参数类型很可能变化）：

### 方法桥接

擦除前：

![](C:\Users\Administrator\AppData\Roaming\marktext\images\2025-05-15-10-42-36-image.png)

擦除后：

![](C:\Users\Administrator\AppData\Roaming\marktext\images\2025-05-15-10-42-48-image.png)

    为了解决这个事情，Java引入了桥接方法，为每个继承/实现泛型类/接口的子类服务，以此保持多态性，字节码如下：

![](C:\Users\Administrator\AppData\Roaming\marktext\images\2025-05-15-10-44-07-image.png)

        其实现原理，就是重写擦除后的父类方法，并在其内部委托了原始的子类方法，巧妙绕过了擦除带来的影响。不仅如此，就算不是泛型类，当子类方法重写父类方法的返回类型是父类返回类型的子类时，编译器也会生成桥接方法来满足重写的规则。

```java
public class Node<T> {
    public T data;
    public Node(T data) { this.data = data; }
    public void setData(T data) { this.data = data; }
}

public class MyNode extends Node<Integer> {
    public MyNode(Integer data) { super(data); }
    public void setData(Integer data) { super.setData(data); }
}

// 编译器会生成桥方法
public void setData(Object data) {
    setData((Integer) data);
}
```

Java核心技术中总结的非常到位：

- 虚拟机中没有泛型，只有普通的类和方法。

- 所有的类型参数都会替换为他们的限定类型。

- 会合成桥接方法来保持多态。

- 为保持类型安全性，必要时会插入强制类型转换。

### 约束与局限性

Java泛型存在一些限制，主要由类型擦除引起：

**1.不能用基本类型实例化类型参数**

基本类型不能直接用作泛型参数，我们只能使用它们的包装类：

```java
Pair<Double> pair = new Pair<>(1.5, 2.5);
```

**2.运行时类型查询只适用于原始类型**

因为类型擦除，所有的类型检查操作只能识别到原始类型：

```java
Pair<Integer> p = new Pair<>(1, 2);
if (p instanceof Pair) { ... }
```

**不能创建参数化类型的数组**

尝试创建参数化类型的数组会导致编译错误，因为这在运行时可能导致类型安全问题：

```java
Pair<Integer>[] table = new Pair<Integer>[10]; // Error
```

**Varargs警告**

使用可变参数时，如果参数是泛型类型，将会收到编译器警告：

```java
public void addAll(Pair<String>... pairs) { ... }
```

可以通过 `@SafeVarargs` 注解来抑制这个警告

**不能实例化类型变量**

我们不能使用 `new T()` 这样的表达式来实例化类型参数：

```java
public static <T> T createInstance(Class<T> cl) {
  return cl.newInstance(); 
}
```

而是需要使用反射或提供一个工厂方法。

>         到这里可以顺便说一下集合的设计，可以注意到集合中只有add方法是泛型参数，而其余方法并不是，为何要这样设计，为何不把其余方法的参数类型也改为E？

![](C:\Users\Administrator\AppData\Roaming\marktext\images\2025-05-15-10-58-11-image.png)

**核心原因：类型擦除的限制**

**`add()` 方法的类型安全**：

- 编译器会在调用 `add()` 时插入 **编译时类型检查**，确保参数类型匹配泛型声明。

- 如果尝试添加错误类型，编译直接报错：

**`remove()` 和 `contains()` 的类型宽松性**：

- 这两个方法的参数类型为 `Object`，因为 **运行时无法获取泛型类型信息**（类型已被擦除）。

- 它们的实现基于 `equals()` 方法，只需判断对象是否逻辑相等，不依赖具体类型：
  
  **设计权衡：向后兼容性**

Java 泛型在 Java 5 引入，设计时必须确保 **新旧代码的兼容性**。在泛型化集合框架时：

**原始类型（Raw Type）的兼容性**：

- Java 5 之前的代码使用原始集合类型（如 `List list = new ArrayList();`）。

- 若将 `remove(Object)` 改为 `remove(T)`，旧代码调用 `list.remove(123)`（传递 `Integer`）会因类型不匹配而无法编译。

- 保持参数为 `Object` 可确保旧代码继续运行。

**API 的二进制兼容性**：

- 修改方法签名（如将 `remove(Object)` 改为 `remove(T)`）会导致已编译的旧版 `.class` 文件无法兼容新版集合框架。

- 保留 `Object` 参数避免了破坏性变更。

## 变型（Variant）与数组

![](D:\Users\Desktop\技术分享\Java泛型\微信图片_20250515092510.png)

​ 变型是类型系统中很重要的概念，主要有三个规则协变，逆变，和不变：

​ 在Java泛型中，协变（covariance）和逆变（contravariance）是用来描述泛型类型参数之间的继承关系如何影响子类型的兼容性。这两个概念主要通过通配符`? extends T`和`? super T`来实现。

### 泛型协变—PECS原则

为了让泛型也支持多态，让其支持协变是很必要的，最常用的场景：我们想让一个方法接受一个集合，并做统一的逻辑处理，如果泛型不支持协变，这种很基本的需求都会成为奢望。

### 协变（Covariance）

协变允许你使用一个比原始指定的派生类型更具体的类型。在Java中，协变通过 `上界通配符 (<? extends T>)` 来实现，这意味着类型参数是T或者是T的子类。协变主要用在返回值上，因为它保证了类型安全，即你不会得到一个不兼容的类型。

例如，假设我们有一个类层次结构：

```kotlin
class Animal {}
class Dog extends Animal {}
```

如果我们有一个方法返回`Animal`类型的对象，我们可以修改这个方法使其返回`Dog`类型的对象，因为`Dog`是`Animal`的子类。在泛型中，我们可以这样写：

```typescript
public List<? extends Animal> getCovariantList() {
    // ...
}
```

这个方法可以返回任何`Animal`类型的列表，包括`Dog`类型的列表。但是，由于我们不知道具体的子类型是什么，我们不能向这个列表中添加任何元素（除了`null`），因为编译器无法确定我们要添加的元素是否与列表的实际类型兼容。

![](D:\Users\Desktop\技术分享\Java泛型\微信图片_20250515092528.png)

### 逆变（Contravariance）

逆变允许你使用一个比原始指定的派生类型更泛化的类型。在Java中，逆变通过`下界通配符 (<? super T>)`来实现，这意味着类型参数是T或者是T的父类。逆变主要用在参数类型上，因为它允许你传递一个更具体的类型给一个期望泛型参数的方法。

例如，如果我们有一个方法接受`Animal`类型的对象：

```typescript
public void addAnimal(List<Animal> animals) {
    // ...
}
```

我们可以修改这个方法使其接受`Object`类型的列表，因为`Object`是所有类的父类。在泛型中，我们可以这样写：

```typescript
public void addContravariantAnimal(List<? super Animal> animals) {
    // ...
}
```

这个方法可以接受任何`Animal`类型的列表，包括`List<Animal>`、`List<Object>`等。由于我们知道列表至少是`Animal`类型，我们可以向这个列表中添加`Animal`或其子类的对象。有利有弊，虽然逆变没有了协变只读不写的限制，但是读取元素时将不能确定具体的类型，只能用Object来接收：

![](D:\Users\Desktop\技术分享\Java泛型\微信图片_20250515092445.png)

### 协变与逆变的区别

- 协变（Covariance）：允许将子类型用作超类型的实例。用于输出（返回值），保证类型安全，但不能写入。
  
  > 例如，`List<? extends Animal>`可以返回任何`Animal`类型的列表。

- 逆变（Contravariance）：允许将超类型用作子类型的实例。用于输入（参数），允许写入更具体的类型，但不保证读取时的类型安全。
  
  > 例如，`List<? super Animal>`可以接受任何`Animal`类型的列表，包括更泛化的类型如`List<Object>`‌

​     

### PECS原则

Producer-Extends, Consumer-Super：

- 当只需要从集合中获取元素（生产者）时，使用`<? extends T>`

- 当只需要向集合中添加元素（消费者）时，使用`<? super T>`

- 当既需要获取又需要添加时，不要使用通配符

![](C:\Users\Administrator\AppData\Roaming\marktext\images\2025-05-15-11-09-12-image.png)

          知道了以上概念，我们需要直接指出，泛型默认是不支持协变的，原因很简单，类型安全：如果允许协变，可能会造成类型转换异常。而数组支持协变，正如文章开头所说，就是设计者希望可以对数组进行比较通用的处理，防止方法为每一种类型编写重复逻辑，这样做也确实导致为数组赋值元素时可能会抛出运行时异常`ArrayStoreException`，这是一个很危险的坑。Effective Java中直接指出允许数组协变是Java的缺陷，我想这也是要多用列表而不用数组的原因之一。

Collections的copy方法就非常好的印证了这一点：

![](C:\Users\Administrator\AppData\Roaming\marktext\images\2025-05-15-11-24-15-image.png)

**数组的协变性示例**

```java
package com.demo.genericity;

public class Test {

    static abstract class Shape {
        public abstract void draw();
    }

    static class Rectangle extends Shape {
        @Override
        public void draw() {
            System.out.println("Drawing a rectangle");
        }
    }

    static class Circle extends Shape {
        @Override
        public void draw() {
            System.out.println("Drawing a circle");
        }
    }

    public static void main(String[] args) {
        Shape[] shapes = new Shape[2];
        shapes[0] = new Rectangle();
        shapes[1] = new Circle();

        for (Shape shape : shapes) {
            shape.draw();
        }
    }
}

Drawing a rectangle
Drawing a circle
```

**Java如何获取运行时泛型的类型**

在Java中，泛型是在编译时实现的，这意味着在运行时，泛型信息会被擦除（即泛型信息不会保留在运行时类型中）。因此，直接从运行时获取泛型的类型是不可能的，就像你不能直接从`List<String>`得到`String`类型一样。

但是，你可以通过以下几种方式在运行时获取或使用泛型类型的信息：

### 1. 使用类型标记（Type Tokens）

类型标记是一种在编译时捕获泛型类型的技术。你可以创建一个辅助类或方法，该类或方法持有对泛型的引用。例如：

```java
public class TypeReference<T> {
    private final Class<T> type;

    public TypeReference(Class<T> type) {
        this.type = type;
    }

    public Class<T> getType() {
        return type;
    }
}
```

使用这个类：

```java
TypeReference<String> typeRef = new TypeReference<>(String.class);
Class<String> type = typeRef.getType();
```

### 2. 使用`java.lang.reflect.ParameterizedType`和`java.lang.reflect.Type`

你可以通过反射获取到泛型类型的信息。例如，如果你有一个方法返回了泛型类型的对象，你可以这样获取其泛型类型：

```java
public static Type getGenericClass(Object object) {
    return object.getClass().getGenericSuperclass();
}
```

然后你可以进一步检查这个`Type`对象是否为`ParameterizedType`，并从中获取实际的类型参数：

```java
public static Class<?> getActualTypeArgument(Object object) {
    Type type = getGenericClass(object);
    if (type instanceof ParameterizedType) {
        ParameterizedType pType = (ParameterizedType) type;
        Type[] typeArguments = pType.getActualTypeArguments();
        if (typeArguments != null && typeArguments.length > 0) {
            return (Class<?>) typeArguments[0]; // 获取第一个类型参数
        }
    }
    return Object.class; // 如果没有找到，返回Object.class作为默认值
}
```

### 3. 使用`java.lang.reflect.Method`和`java.lang.reflect.Type`（对于方法返回类型）

如果你知道某个方法返回了泛型类型的对象，你可以通过反射获取这个方法的返回类型，并进一步分析它是否是参数化类型：

```java
public static Type getGenericReturnType(Method method) {
    return method.getGenericReturnType();
}
```

### 4. 使用第三方库如Google的Guava库中的`TypeToken`类

Google的Guava库提供了一个名为`TypeToken`的类，它可以帮助你以类型安全的方式处理泛型：

```java
TypeToken<List<String>> typeToken = new TypeToken<List<String>>() {};
Class<? extends List> rawType = typeToken.getRawType(); // 获取原始类型 List.class
Type[] actualTypeArguments = typeToken.getType().getActualTypeArguments(); // 获取泛型参数 String.class
```

### 原理分析

**JVM的ClassFile标准**

JVM的ClassFile就是Java源文件编译后产生的二进制格式。类似于Linux下的ELF或者Windows的COFF，可以简单的理解为JVM的“可执行文件”。JVM通过读取它，并执行bytecode，最终执行程序的运行。ClassFile的格式如下:

![](C:\Users\Administrator\AppData\Roaming\marktext\images\2025-05-28-20-38-57-image.png)

其中的attributes数组里，就是JSR14里提到的，泛型信息的保存所在。JVMS指出[11]

> A Java compiler must emit a signature for any class, interface, constructor, method, or field whose declaration uses type variables or parameterized types 

可以看到Java编译器需要把泛型类信息带到Signature这个attribute，然后存储于编译后的ClassFile里。



[Java中如何获得A<T>泛型中T的运行时类型及原理探究](https://mp.weixin.qq.com/s/Yn9CIfgLozZNs_xfSo1ZmA)
