Manifold是一个Java编译器插件，通过在__编译期__生成代码来增强Java的能力，它提供了若干模块，每个模块可以独立依赖。

# 1 core

`Type Manifold` 通过`Javac plugin api`直接插入Java编译器，作为通用类型适配器，允许直接无缝地提供Java类型系统无法访问的类型和功能。该框架的核心库提供了一个基础和插件SPI来动态解析类型名称并按需生成Java源代码，以更普遍地增强Java的类型系统。实现SPI的插件称为`Type Manifold`。

`Type Manifold`可以提供与任何类型化数据对应的新类型，例如JSON、XML、GraphQL甚至JavaScript等语言。此外，`Type Manifold`还可以通过附加方法、属性、接口、注释等来增强现有类型。

Manifold框架可以作为即时代码生成器。Manifold框架通过覆盖编译器的类型解析器，使用ITypeManifold SPI来声明对类型名称的所有权，并动态提供与类型对应的源代码。与传统的代码生成技术相比，Manifold框架可以在编译器需要类型时按需生成代码，从而大大减少了代码生成的复杂性，并使其能够以增量方式运行。此外，Manifold框架还可以从IDE和其他工具中使用，以提供对所有类型和功能的一致、统一的访问。

Manifold 包括静态(编译期)和运行时，推荐使用静态，所有的依赖都添加 `scope = provided`。

# 2 扩展

通过 `manifold-extends` 模块可以给现有类添加新的方法，它本质上是在编译器把在实例上的方法调用转换成对静态方法的调用。

为了使用 `manifold-extends` 需要添加依赖和编译插件:

```xml
 <dependency>
    <groupId>systems.manifold</groupId>
    <artifactId>manifold-ext</artifactId>
    <version>${manifold.version}</version>
    <scope>provided</scope>
</dependency>
        
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.8.1</version>
    <configuration>
        <source>8</source>
        <target>8</target>
        <encoding>UTF-8</encoding>
        <compilerArgs>
            <arg>-Xplugin:Manifold no-bootstrap</arg>
        </compilerArgs>
        <annotationProcessorPaths>
            <path>
                <groupId>systems.manifold</groupId>
                <artifactId>manifold-ext</artifactId>
                <version>${manifold.version}</version>
            </path>
        </annotationProcessorPaths>
    </configuration>
</plugin>
```

## 2.1 扩展方法

原则：

1. 扩展方法的优先级低于`class method`: 扩展方法只能提供新能力，不能覆盖已有能力(注意方法签名，签名一致才是同一个方法)

#### 2.1.1 扩展实例方法

```java
@Extension
public class ObjectExt {

  public static Optional<@Self Object> asOpt(@This Object obj) {
    return Optional.ofNullable(obj);
  }
}
```

可以发现本质上还是工具类的静态方法，但是有一些要求：

1. 工具类需要使用 Manifold 的 @Extension 注解
2. 静态方法不能是 `private`，静态方法中，目标类型的参数，需要使用 @This 注解，标明它是方法的 `receiver`
3. 工具类所在的包名，需要以 **extensions.目标类型全限定类名** 结尾(模仿的 C# 的扩展方法)。

#### 2.1.2 扩展静态方法

静态方法没有 `receiver`，需要在方法上添加 `@Extension` 注解

```java
 @Extension
  public static boolean eq(Object o1, Object o2) {
    return Objects.equals(o1, o2);
  }


  @Extension
  public static boolean neq(Object o1, Object o2) {
    return !eq(o1, o2);
  }
```

这里提到了 `@ThisClass`注解： Sometimes it is useful to define a static extension method in a base class that knows which derived class called the method (the "receiver" of the call).

#### 2.1.3 扩展内部类的方法

扩展内部类的方法时，扩展对象的结构要和内部类的结构一致，在外部类上打`Extension`注解，然后再外部扩展类中创建同名内部类:

```java
@Extension
public class MapExt {
    public static <K,V> String myMapMethod(@This Map<K,V> thiz) {
        return "myMapMethod";
    }

    public static class Entry {
        public static <K, V> K key(@This Map.Entry<K, V> e) {
            return e.getKey();
        }

        public static <K, V> V value(@This Map.Entry<K, V> e) {
            return e.getValue();
        }
    }
}
```

#### 2.1.4 扩展数组

mainifold 提供了一个现成的 `ManArrayExt`，关于数组的部分可以参考它的写法，它提供了如下的能力:

```java
  List<@Self(true) Object> toList()
  boolean isEmpty()
  boolean isNullOrEmpty()
  @Self Object copy()
  @Self Object copy(int newLength)
  @Self Object copyTo(@Self Object to)
  @Self Object copyRange(int from, int to)
  @Self Object copyRangeTo(int from, int to, @Self Object target, int targetIndex)
  Stream<@Self(true) Object> stream()
  void forEach(IndexedConsumer<? super @Self(true) Object> action)
  Spliterator<@Self(true) Object> spliterator()
  int binarySearch(@Self(true) Object key)
  int binarySearch(int from, int to, @Self(true) Object key)
  int binarySearch(@Self(true) Object key, Comparator<? super @Self(true) Object> comparator)
  int binarySearch(int from, int to, @Self(true) Object key, Comparator<? super @Self(true) Object> comparator)
  int hashCode()
  boolean equals(@Self Object that)
```

注意 `@Self` 注解的使用，它提供对数组的组件类型和数组类型本身的类型安全访问。

同时注意使用 `Object` 类型而不是数组类型来支持引用数组和基本数组的类型推断

## 2.2 扩展属性

略，感觉不合适。

## 2.3 扩展接口

还没完全看明白

## 2.4 官方实现的扩展

重点看看collection

### 2.5 类型安全的反射调用 `@Jailbreak`

有时候必须要通过反射获取私有属性或者通过反射访问私有方法，在写这些代码时既麻烦又容易出错，manifold 提供了一种类型安全的写法，它本质上还是在编译器把反射操作替换成它自己的反射工具类。

__注意__: `Jailbreak` 提出的动机是为了方便在 `test` 中使用。

###### 2.5.1 访问实例属性和方法

比如我们有一个类，它有一个私有常量字段和一个私有方法:

```java
public class Foo {
		private final int privateField;

    public Foo(int value) {
        privateField = value;
    }

    private String privateMethod() {
        return "hi";
    }
}
```

如果我们想访问属性和方法只能通过反射，现在 `manifold` 来了:

```java
// 使用 @Jailbreak 来注解实例
@Jailbreak Foo accessor = new Foo(10);
// 可以给属性赋值
accessor.privateField = 88;
// 可以调用私有方法
accessor.privateMethod();

Foo normal = new Foo(20);
// 既不能赋值属性也不能调用方法
normal.privateField = 88; // compile error
normal.privateMethod(); // compile error
```

本质上在编译器把上述操作转换成了 它自己的反射工具调用：

```java
ReflectionRuntimeMethods.setField_int(foo, "privateField", 88);
ReflectionRuntimeMethods.invoke_Object(foo, "privateMethod", new Class[0], new Object[0])
```

2.5.2 访问静态属性和方法

对于静态属性和静态方法，仅仅需要一个类型标识就行，不需要创建实例:

```java
class Foo {
    private static String greeting = "hi";

    private static void sayGreeting() {
        System.out.println(greeting);
    }
}

@Jailbreak Foo staticFoo = null;
staticFoo.greeting = "Hello"; --> ReflectionRuntimeMethods.setFieldStatic_Object(Foo.class, "greeting", "Hello");
staticFoo.sayGreeting(); --> ReflectionRuntimeMethods.invokeStatic_Object(Foo.class, "sayGreeting", new Class[0], new Object[0]);
```

###### 2.5.3 访问non-public class

假设我们有一个包可见的类:

```java
package manifold.jailbreak.a.b.c;

class SecretClass {
    private final String _data;

    // not public
    SecretClass(String data){
        _data = data;
    }

    private void callme() {
        System.out.println("callme:  " + _data);
    }
}
```

如何在包外访问它呢?

```java
manifold.jailbreak.a.b.c. @Jailbreak SecretClass secret = 
										new manifold.jailbreak.a.b.c. @Jailbreak SecretClass("Jailbreak");
secret.callme();
```

它同时提供 `jailbreak()` 的扩展方法，不过在 `IDEA 2021.3` 插件中支持不好。

# 3 操作符重载

# 4 structural(依赖manifold运行时)

让Java实现了【鸭子类型】：

1. 一个对象有效的语义，不是由继承自特定的类或实现特定的接口，而是由“当前[方法](https://zh.wikipedia.org/wiki/方法_(電腦科學))和属性的集合”决定
2. 当看到一只鸟走起来像鸭子、游泳起来像鸭子、叫起来也像鸭子，那么这只鸟就可以被称为鸭子。

比如我们定义如下的接口：

```java
@Structural
public interface Coordinate {
    double getX();
    double getY();
}
```

实现基于 `Coordinate`的比较器:

```java
Comparator<Coordinate> coordSorter = (c1, c2) -> {
            int x = Double.compare(c1.getX(), c2.getX());
            int y = Double.compare(c1.getY(), c2.getY());
            return x != 0 ? x : y;
        };
```

那么如果一个类，它有 `getX()` 和 `getY()` 的方法，并且这两个方法的返回值可以兼容 `double`，那么 `manifold` 就可以让它看起来是 `Coordinate`:

```
@AllArgsConstructor
@Getter
public class Point {
    private int x;
    private int y;
}
```

那么 `Point` 类就可以直接使用 `coordSorter`:

```java
List<Point> points = Arrays.asList(
                new Point(1, 2),
                new Point(1, 3),
                new Point(2, 3),
                new Point(2, 4));
Collections.sort((List) points, coordSorter);
```

通过反编译可以看到利用了反射在运行时动态获取对象的 `getX()` 和 `getY()` 的值:

```java
Comparator<Coordinate> coordSorter = (c1, c2) -> {
            int x = Double.compare(((Coordinate)RuntimeMethods.constructProxy(c1, Coordinate.class)).getX(), ((Coordinate)RuntimeMethods.constructProxy(c2, Coordinate.class)).getX());
            int y = Double.compare(((Coordinate)RuntimeMethods.constructProxy(c1, Coordinate.class)).getY(), ((Coordinate)RuntimeMethods.constructProxy(c2, Coordinate.class)).getY());
            return x != 0 ? x : y;
        };
```

`structural` 还有其他功能，



