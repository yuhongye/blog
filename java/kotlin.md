# 1 语法

### 基础

1. 数值类型没有自动提升，也就是一个函数的参数是double，则不能传入flout或者long，虽然缺少隐式转换，但是在运算中会有重载做适当的转换: `val x = 1L + 3 // Long + Int => Long`

2. 如果一个类有主构造函数，次构造函数必须调用主构造函数，初始化语句块是属于主构造函数的一部分，主构造函数不能有任何语句，因此属性初始化和初始化块组成了主构造函数的全部语句，按顺序执行，也就可以理解为：所有的类都有一个主构造函数，如果是显式指定了，则次构造函数也必须显式委托。在JVM机器上，如果主构造函数的全部属性都有默认值，则会额外生成一个无参数的主构造函数，其属性全部是默认值，像是一个语法糖。

3. kotlin没有`new`关键字

4. 所有类都是`Any`的子类，有三个方法: `equals`,`hashCode`,`toString`，默认情况下Kotlin的类都是`final`的，必须要显式的指定`open`才能允许继承，`open class Base`

5. 父类允许被覆盖的函数也必须要加上`open`，并且子类的方法要显式加上`override`，否则不允许覆盖。在子类中如果覆盖了父类的函数，但是又不允许它的子类继续覆盖，则加上`final`关键字

   ``` 
   open class Shape {
   	open fun draw() { // 允许覆盖 }
   	fun fill() { // 不允许覆盖}
   }
   
   open class Circle(): Shape() {
   	final override fun draw() { // 覆盖了父类的方法，但是不允许子类继续覆盖 }
   }
   ```

6. 父类的`val`属性可以被子类的`var`覆盖，反之则不行。那这样的话，父类里的方法怎么对待这个属性，当成val还是var，这在多线程中很有关系

7. fun 关键通过带有`=`来判断是普通函数体还是表达式函数体

8. 如果函数的参数不止一个，且最后一个参数是函数类型时，可以使用柯理化风格来调用

   ```java
     fun curryingLike(content: String, block: (String) -> Unit) {
           block(content)
      }
   val curring = curryingLike("caoxiaoyong") {c -> println(c)}
   val curring = curringLike("错误的，Kotlin没有实现真正的柯理化，不能这么调用")
   ```

9. by lazy可已经进行延迟初始化，同时可以指定线程安全级别等级; lateinit可以用来对var进行延迟初始化，但是不能用于primitive type

   ```kotlin
   val sex: String by lazy(LazyThreadSafetyMode.PUBLICATION)
   lateinit var set: String
   ```

10. Kotlin中所有非抽象属性在定义的时候必须有默认值