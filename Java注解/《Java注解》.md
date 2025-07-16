# Java注解(Annotation)

> Java注解是Java5. 0版本开始支持特殊语法的元数据，Java中的类、方法、变量、参数都可以通过注解进行标记，并可以通过反射方式获取注解的元数据信息。

## 为什么需要注解？

### 1. 元数据管理的本质需求

#### 问题背景

在注解出现之前，Java 的元数据主要通过以下方式表达：

```xml
<!-- Spring的Bean配置 -->
<bean id="userService" class="com.example.UserService">
    <property name="timeout" value="1000"/>
</bean>
```

- **痛点**：配置与代码分离，维护困难，易出错。

**命名规范/注释**：

```java
// @Transactional 的早期替代方案
public class UserService {
    // 方法名前缀表示事务性
    public void tx_updateUser() {...}
}
```

- **痛点**：无编译器校验，易被忽略。

#### **注解的解决方案**

注解将元数据**内联到代码中**，形成自描述的编程模型：

```java
@Transactional(timeout = 1000)
public void updateUser() {...}
```

**优势**：

- 代码与配置合一，减少维护成本。
- 编译器/工具链可直接处理注解（如`@Override`的语法检查）

### 2. 注解的核心价值

#### **(1) 标准化元数据表达**

- **统一格式**：注解提供了一种类型安全、结构化的元数据格式，替代松散的注释或配置文件。
- **编译时校验**：注解可被编译器检查（如`@Deprecated`触发警告）。

#### **(2) 支持声明式编程**

- **框架设计的革命**：
  - Spring 的 `@Component`、`@Autowired` 取代XML配置。
  - JPA 的 `@Entity`、`@Column` 替代ORM映射文件。
  - **底层机制**：反射读取注解，动态生成行为（如Spring扫描`@Component`创建Bean）。

#### **(3) 编译时/运行时动态处理**

- **编译时处理（APT）**：
  - 通过`javax.annotation.processing`生成代码（如Lombok的`@Data`）。
  - **案例**：`@Getter`注解自动生成getter方法字节码。
- **运行时反射**：
  - 框架通过反射获取注解信息（如Spring MVC的`@RequestMapping`路由映射）。

> 注解的使用可以简化配置信息，易于维护代码，可读性强，同时符合代码约定大于配置原则（Convention over Configuration）

## 注解的介绍

### 注解的典型应用场景

| 场景        | 代表注解                     | 作用              |
| --------- | ------------------------ | --------------- |
| **框架配置**  | `@SpringBootApplication` | 启动Spring Boot应用 |
| **依赖注入**  | `@Autowired`             | 自动装配Bean        |
| **AOP切面** | `@Aspect`                | 声明切面类           |
| **测试**    | `@Test`                  | JUnit测试方法标记     |
| **代码生成**  | `@Builder`               | Lombok生成建造者模式代码 |
| **序列化控制** | `@JsonIgnore`            | Jackson忽略字段序列化  |

**Java内置注解**

| @Override          | 表示继承和覆写       |
| ------------------ | ------------- |
| @Deprecated        | 表示废弃          |
| @SuppressWarnings  | 表示压制警告        |
| @SafeVarargs       | 不会对不定项参数做危险操作 |
| @FunctionInterface | 声明功能性接口       |

**JDK预定义元注解**

| @Target     | 设置目标范围 |
| ----------- | ------ |
| @Retention  | 设置保持性  |
| @Documented | 文档     |
| @Inherited  | 注解继承   |
| @Repeatable | 重复修饰   |

@Retention 元注解可以指定一个注解的生命周期。它有三个取值：

- `RetentionPolicy.SOURCE`：注解在源码，不在class中。注解将被编译器丢弃。（只会保留到 java 源代码阶段在编译期就会被编译器抛弃，所以通过反射获取返回 null）如：@Override
- `RetentionPolicy.CLASS`：注解在源码和class文件中，但在 JVM 运行时不会被保留。（只会保留到编译期也就是 class 文件中，但是在运行时会被 JVM 抛弃，所以这里通过反射获取也是 null）
- `RetentionPolicy.RUNTIME`：注解在源码和class文件中，在 JVM 运行时也会被保留，所以可以通过反射获取。

@Target 元注解可以指定一个注解可以应用的程序元素类型。其接受一个 `ElementType` 的枚举类型数组，比如可以指定一个注解只能应用于方法、类、变量等。

```java
public enum ElementType {
    /** Class, interface (including annotation type), or enum declaration */
    TYPE,

    /** Field declaration (includes enum constants) */
    FIELD,

    /** Method declaration */
    METHOD,

    /** Formal parameter declaration */
    PARAMETER,

    /** Constructor declaration */
    CONSTRUCTOR,

    /** Local variable declaration */
    LOCAL_VARIABLE,

    /** Annotation type declaration */
    ANNOTATION_TYPE,

    /** Package declaration */
    PACKAGE,

    /**
     * Type parameter declaration
     *
     * @since 1.8
     */
    TYPE_PARAMETER,

    /**
     * Use of a type
     *
     * @since 1.8
     */
    TYPE_USE
}
```

TYPE：作用为类

FIELD：作用属性

METHOD：作用方法

PARAMETER：作用方法参数

CONSTRUCTOR：作用构造方法

LOCAL_VARIABEL：作用本地变量

PACKAGE：作用于包

TYPE_PARAMETER：1.8 提供 作用于泛型参数

TYPE_USE：表示可以作用在包和方法除外的任何类型

@Inherited 允许被子类继承的注解，注解默认是不能被继承的，但是在使用 Inherited 注解后子类可以继承父类的注解。

@Repeatable主要用于解决一个类或方法上不能标注重复的注解的问题。在Java 8之前，每个注解只能应用一次，如果需要应用多个相同的注解，需要使用容器注解来实现，这会使代码变得复杂且难以维护。@Repeatable注解的引入简化了这一过程，使得注解可以在同一个地方重复使用‌

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Repeatable(value = MyAnnotationList.class)
public @interface MyAnnotation {
    String name();
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface MyAnnotationList {
    MyAnnotation[] value();
}

public class MyAnnotationTest {

    // 本质上 这里是隐藏了@MyAnnotationList容器注解 实际上是多个注解都会塞进@MyAnnotationList容器注解中
    @MyAnnotation(name = "Test1")
    @MyAnnotation(name = "Test2")
    private void testMethod() { }

    @MyAnnotationList({@MyAnnotation(name = "Test1"), @MyAnnotation(name = "Test2")})
    private void testMethod2() { }
}
```

### 注解的底层实现原理

> 注解本质是一个继承了Annotation的特殊接口，其具体实现类是Java运行时生成的动态代理类。通过代理对象调用自定义注解（接口）的方法，会最终调用`AnnotationInvocationHandler`的invoke方法。该方法会从`memberValues`这个Map中索引出对应的值。而`memberValues`的来源是Java常量池。

#### **(1) 注解的本质**

- **接口的派生类型**：所有注解继承`java.lang.annotation.Annotation`，编译后为接口。

```java
// 源码
public @interface Transactional {...}

// 编译后（反编译）
public interface Transactional extends Annotation {...}
```

#### **(2) JVM层面的存储**

**字节码属性表**：注解信息存储在Class文件的`RuntimeVisibleAnnotations`属性中。

- 通过`javap -v`查看字节码：

```java
RuntimeVisibleAnnotations:
  0: #10(#11=I#12)
    @Transactional(timeout=1000)
```

#### **(3) 运行时读取机制**

**反射API**：通过`getAnnotation()`读取注解时，JVM动态生成**代理类**。

```java
Transactional anno = method.getAnnotation(Transactional.class);
// 实际调用的是JVM生成的代理类
class $Proxy1 implements Transactional {
    public int timeout() { return 1000; }
}
```

### **注解的潜在问题与规避**

- **过度使用**：滥用注解会导致代码可读性下降（如“注解地狱”）。
- **性能开销**：运行时反射解析注解有轻微性能损耗（可通过缓存优化）。
- **模块化限制**：JDK9+需`opens`包才能反射访问注解。

### **总结：注解的哲学意义**

Java 引入注解，实质上是**将程序的“做什么”（What）与“怎么做”（How）解耦**：

- **开发者**通过注解声明意图（如`@Test`）。
- **框架/工具**负责实现具体逻辑（如JUnit运行测试）。

这种约定优于配置（Convention over Configuration）的思想，极大提升了开发效率，成为现代Java生态的基石。


