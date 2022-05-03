使用Scala有一年了，之前只是粗略的浏览了《Scala编程》前面的章节，导致在日常开发中写的代码不够优雅，经常会在一起问题上卡壳，开发效率也不够高。基于自己熟练掌握Java和能使用Scala进行命令式编程的基础上重新来学习Scala。

### 0. Scala实用指南

### 1. 从Java到Scala

1. 所有Java基本类型在Scala包中都有对应的类，scala.Boolean对应Java的boolean, scala.Int对应Java的int. Scala编译器会尽量使用Java的基本类型，让你的代码享受基本类型的性能优势。TODO: 什么情况下不会编程成基本类型呢?

   ```
   // 涉及到泛型的时候会编程包装类型
   val t3 = ("cxy", 1, 10000)
   // 翻译成Java代码后
   Tuple3 t3 = new Tuple3("cxy", BoxesRunTime.boxToInteger(1), BoxesRunTime.boxToInteger(10000));
   ```

   

2. 在 Scala 中，如果方法没有参数或者只有一个参数，可以省略点号和括号，如果一个方法带多个参数，则必须使用括号，但是点号仍然可以省略。

3. `for (i <- 1 to 3)` 中的 `i` 是 `val`，在每次循环时都会创建一个不同的 `val i`

4. scala 参数默认值，def f(a: Int = 1, b: Int = 2, c: Boolean = false)，如果对某个参数使用了默认值，则它后面所有的参数都得使用默认值

5. 命名参数: def f (a: Int = 1, b: Int = 2) 可以这样调用: f(a = 2, b = 1)

6. 隐式参数，默认参数值是由函数定义者指定的，如果由调用者指定默认参数呢？隐式参数可以达到这一目的:

   ```scala
   case class Wifi(name: String)
   
   def connectTo(user: String)(implicit wifi: Wifi) = {
     println(s"$user connect to $wifi")
   }
   
   def main(args: Array[String]): Unit = {
     implicit def office: Wifi = new Wifi("office")
     
     connectTo("cxy") // 将会打印: cxy connect to office
     connectTo("cxy")(Wifi("home")) // 将会打印: cxy connect to home
   }
   ```

   使用隐式参数的注意点：

   1. 隐式参数要放在一个单独的参数列表中
   2. 在一个作用域中，对于同一类型只能有一个隐式变量，否则编译器会报错

   

   隐式参数。

7. scala 没有操作符，scala 通过使用函数名的第一个字符作为优先级判定

8. `==` 在Java中 `==` 对于引用类型比较的是它们的内存地址，而在 Scala 中 `==`调用的是`equals()`，这样原始类型和引用类型就统一了，都是比较值。如果想比较引用地址，则需要使用`eq`方法

9. 访问修饰符

   a. 默认类、成员变量和方法都是 public

   b. `protected` 仅对自己和派生类可见，而Java 则对同一个包下可见

   c. `private` 默认是类级别可见，但是 Scala 提供了细粒度控制:

   ```scala
   private[Class_A] val b = 1; // b 对于 Class_A 是可见的
   private[packge_b] val c = 1 // c 对于 package_b 包下的所有类都是可见的
   private[this] val x = 2 // x 仅对该实例可见，同一类下的不同实例都不可见
   
   // 讲一个类的主构造器设置成 private
   class Dude private(var firstName, val secondName)
   ```

### 2. Scala 对象系统

1. Scala 强制规定：辅助构造器的第一行有效语句必须调用主构造器或者其他辅助构造器

2. 在定义 var 字段时如果要赋默认值可以使用如下语句: private var name: String = _

3. Scala 中会对字段生成 getter 和 setter 方法

   ```scala
   class Dude(val firstName: String, private val secondName: String, var pv)
   ```

   在上面的类中等价的 Java 代码如下:

   ```java
   public class Dude {
     private final String firstName;
      private final String lastName;
      private int age;
   
     // firstName 的 getter 方法，注意 Scala 默认是 public 的
      public String firstName() {
         return this.firstName;
      }
   
     // second 的 getter 方法，注意 lastName 的访问控制修饰符是 private
     // 所以这里的 getter 也是 private
      private String lastName() {
         return this.lastName;
      }
   
      public int pv() {
         return this.age;
      }
   
     // pv 是 var 类型，所以给它生成了 setter 方法: pv_=
      public void pv_$eq(int x$1) {
         this.age = x$1;
      }
   
      public Dude(String firstName, String lastName, int age) {
         this.firstName = firstName;
         this.lastName = lastName;
         this.age = age;
         super();
      }
   }
   ```

   当时 Scala 生成的方法不符合 JavaBean 的规范，如果想要符合 JavaBean 的规范可以添加 `@BeanProperty`注解，这样编译器就会同时生成  JavaBean 和 Scala 风格的 getter

   ```scala
   class Dude(@BeanProperty val firstName)
   ```

   生成的Java代码

   ```java
      private final String firstName;
      
   	// scala 风格的 getter
      public String firstName() {
         return this.firstName;
      }
      // JavaBean 的 getter
      public String getFirstName() {
         return this.firstName();
      }
   ```

4. 别名: type MSet = mutable.HashSet

5. 扩展一个类，在Scala中扩展一个基类和Java很像，只是增加了两个非常好的限制：

   1. 方法的重载必须使用 `override` 关键字
   2. 只有主构造器能传递参数给基类的构造器 `class Sub extends Base(args)`

6. 创建枚举类

   ```scala
   object Device extends Enumeration {
     // 告诉编译器将 Device 视为一个类型而不是一个实例
     // 注意我们使用 object 关键字创建了一个单例对象，所以要增加这个怪语法
     // scala 后续版本会改变枚举的定义
     type Device = Value
     val PC, WISE = Value
   }
   ```

7. 包对象： Scala 中包除了可以定义类、接口、枚举和注解类型外，还可以定义变量和函数，它们被放在一个成为 package object 的特殊的__单例对象__中。如果你发现自己创建一个类仅仅是为了在同一个包中共享的static 方法，那么应该把这些方法放到 package object 中

8. Either类型有两个子类: Right 和 Left，通常使用Right表示正常只，Left表示错误。如果想要表明一个值可能不存在则使用`Option`，如果结果可能在两个不同的值之间变化，就使用 `Either`，感觉有点像C语言中的union

   ```scala
   //                             left      right 
   def either(input: Int): Either[Double, String] = {
       if (input > 0) {
         Right("Positive")
       } else {
         Left(-1.0)
       }
     }
   
   ```

9. 函数返回值的类型推断：只有使用 `=` 将方法的声明和方法主体区分开时才会自动推断。

   ```scala
   def function1 { Math.sqrt(4) } // function1 返回的值类型是 Unit
   def function2 = Math.sqrt(4)   // function2 返回值的类型是 Double
   ```

10. 协变：期望接受一个基类实例的集合的地方，能够使用一个子类实例的集合的能力叫做协变。

11. 逆变：期望接受一个子类实例的集合的地方，能够使用一个基类实例的集合的能力叫做逆变

12. 默认情况下，Scala即不允许协变，也不允许逆变

13. 在Scala中如果要支持协变，需要明确告知: `T <: BaseClass`，表明 T 派生自 BaseClass，它指明了上界，也就是BaseClass。在定义集合是如果要支持协变，可以通过使用`[+T]`来指明

14. 在Scala中如果要支持逆变，需要明确告知. 在定义泛型时可以使用[-T]来指明

    ```scala
      /**
       * @param from 子类型的数组
       * @param to 父类型的数组
       */
      def copy[SUB, BASE >: SUB](from: Array[SUB], to: Array[BASE])
    ```

15. 隐式方法：implicit def convertXToY(x: X): Y = { y }

16. 隐式类：相比于隐式方法更推荐隐式类，饮食类有一些限制：它不能是一个独立的类，它必须要在一个单例对象、类或者特质中。隐式类的定义: `implicit class X`

17. 值类，继承自 `AnyVal`，Scala会尽量避免不生成对象，这样在字节码层面值类就是它持有的值，因此这也限制了值类只能有一个字段。

    ```scala
    class X(val id: Int) extends AnyVal
    这个类在空间占用上等同于 Int ，同时它又可以扩展 Int 的表现形式
    
    val v1 = new X(1) // 不会创建对象
    val v2: Any = new X(2) // 会创建对象
    ```

    