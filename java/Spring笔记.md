这篇文档不是一蹴而就的，而是反复迭代，开始时只掌握其最常用、最简单的用法，随着学习的逐步深入，会把有必要的东西不断地补充进来。

# 1 The Spring context: Defining beans

在使用Spring时最重要的是就让Spring去管理 beans，Spring 使用 the context(also known as the application context in spring app) 来管理bean。因此这里就两件最基本的事情：

1. 如何定义bean
2. 如何把bean加入到 spring context 中，spring context 只有知道了某个 bean 的存在才能去管理它。

额外需要说明的是：不是所有的对象都需要放到 spring context 中去的。

#### 1.1 ApplicationContext

什么是 spring context ?  the place in the app's memory where we add the object instances we want spring to manage。Spring 提供了多种 `ApplicationContext` 的实现，在当前最常使用的就是 `AnnotationConfigApplicationContext`, 可以理解成所有的`bean`都是由它来管理的。

在 Spring 5.0 以后提供了三种方式来把 bean 添加到 spring context 中。

#### 1.2 使用@Bean 注解

在这种方式中把 bean 放到 spring context 中需要做两件事：

1. 使用 `@Bean` 注解来定义 bean
2. 需要一个Config Class 来管理 bean 的定义：可以理解成 Spring Context 只会扫描 Config Class 中的 bean 定义

```
// 告诉 spring context 这是一个配置类
@Configuration
public class ProjectConfig {

	/**
	 * 定义一个bean，bean的名称默认是方法名
	 */
	@Bean
	public String assetName() {
		return "asset_1";
	}
}

public class Main {
	public static void main(String[] args) {
	  // 在初始化 spring context 时指定 config classes: 注意这是一个变长参数
		var ctx = new AnnotationConfigApplicationContext(ProjectConfig.class, other config class);
		// 它们俩代表的是同一个 bean
		val assetFromType = ctx.getBean(String.class);
		val assetFromName = ctx.getBean("assetName", String.class);
	}
}
```

`@Bean`可以定义同一种类型的多个bean实例，下面会演示几种用法:

```
public class IdGenerator {
	private int nextId;
	// 省略构造方法 和 getter setter
}

@Configuration
public class ProjectConfig {

	// 定义一个bean，bean的名称默认是方法名
	@Bean
	public IdGenerator threadIdGenerator() {
		return new IdGenerator(1);
	}
	
	// 显式的给bean一个名字
	@Bean(name = "db-connection-id-generator")
	public IdGenerator xIdGenerator() {
		return new IdGenerator(1);
	}
	
	// @Bean 的 name 和 value 属性是一个数组，因此一个 bean 可以有多个名字
	@Bean({"db-connection-id-generator", "db-c-id"})
	public IdGenerator xIdGenerator2() {
		return new IdGenerator(1);
	}
	
	/**
	 * 在使用 ctx.getBean(IdGenerator.class) 时如果有多个同类型的 bean，那么 spring 是不会擅作主张来决定选哪一个的，
	 * spring 如果无法明确的判断该返回哪个 bean，它会抛异常。
	 * 可以使用 @Primary 来帮助 spring 判断：当不指定 bean name 时就返回它。
	 */
	 @Bean
	 @Primary
	 public IdGenerator commonIdGenerator() {
	 		return new IdGenerator(1);
	 }
	 
	 /**
	  * @Bean 最强大的功能就是我们可以返回任意的对象，而不仅仅是我们应用自己定义的对象
	  */
	 @Bean
	 @Primary
	 public Integer defaultNextId() {
	   return 0;
	 }
	 
	 
	 @Bean
	 public Integer dbNextId() {
	   return 1001;
	 }
	 
	 /**
	  * 使用@Bean注解，spring context 就会接管这个bean的创建，spring context 意识到我们需要一个 Integer 类型的 bean,
	  * 它就会去自己的容器里找 Integer 类型的类，如果不发生错误的话就会把它作为参数注入进来, 在这个例子中会把defaultNextId()返回
	  * 的bean注入进来，因为它是primary
	  */
	 @Bean
	 public IdGenerator commonIdGenerator(Integer startId) {
	 		return new IdGenerator(startId);
	 } 
	 
	 /**
	  * 当然我们可以使用名称限定来指定具体使用哪个bean
	  */
	 @Bean
	 public IdGenerator dbIdGenerator(@Qualifier("dbNextId") Integer startId) {
	 		return new IdGenerator(startId);
	 }   
}
```

#### 1.3 使用 stereotype annotations 来添加 bean

使用注解来添加 bean 也需要 spring context 知道两件事：

1. 哪些类需要作为 bean 来添加：使用 `@component` 注解告诉 spring context 初始化这个类的实例，并把它作为 bean 管理起来
2. 需要知道扫描哪些包来自动生成bean，因为不能把全部的包都扫描了，这样性能会慢更重要的是会带来逻辑问题：使用 `@ComponentScan` 来告诉扫描的包名。注意：这个注解是用在 Config Class 身上的

```
package com.cxy.hll;

/**
 * 可以显式的指定bean的名称，默认情况 bean name 是首字母小写，如果开头连续的字母都是大写情况则又不同。
 * 使用 Component 要求类有默认的无参构造方法
 */
@Component("hll")
public class HLL {
	private int precision;
	
	/**
	 * 有三点值得注意：
	 * 1. 由于使用 @Component 是 bean 的创建是由 spring context 来管理的，因此它提供了一种当bean创建完成后还能让我们做点什么的方式，就是用@PostConstruct注解
	 * 2. @PostConstruct 注解的方法不能有参数，也就是 spring context 无法自动给它注入参数：todo 真是这样吗?
	 * 3. 使用 @Bean 创建的 bean 也会执行 @PostConstruct 注解的方法
	 */
	@PostConstruct
	public void init() {
		precision = 14;
	}
}
```

```
// 在 config class 上指明需要扫描的包
@Configuration
@ComponentScan(basePackages = {"com.cxy.hll", other packages})
public class ProjectConfig {

	/** @Component 注解的使用不影响 @Bean 注解 */
	
	@Bean
	@Primary
	public HLL defaultHLL() {
		return new HLL(16);
	}
}
```

下面总结一下使用 `@Bean` 和 `@Component`的区别

| 场景               | @Bean                                                        | @Component                                                   |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| bean创建的控制     | 全部的控制权限，怎么初始化怎么设置参数完全由你来控制，spring只管把创建后的对象拿过来放到自己的容器 | bean的创建由spring控制，spring预留了一个@PostConstruct注解来让你实现对象创建后的动作 |
| 能否创建多个bean   | 是                                                           | 否                                                           |
| 可以创建的bean范围 | 全部的类                                                     | 只有 自己应用程序创建的类，因此需要在类的定义出添加注解。    |
| 便利程度           | 需要写样板代码，只有特殊情况下才使用这种形式                 | 优先选择                                                     |

#### 1.4 手动的添加 bean 到 spring context

在 spring 5.0 以后提供了手动添加 bean 到 spring context 的方式，这意味我们可以在程序运行时动态的去改变 spring context 里的内容。提供的主要方法是：

```
public <T> void registerBean(@Nullable String beanName, Class<T> beanClass,
      @Nullable Supplier<T> supplier, BeanDefinitionCustomizer... customizers);
1. beanName: 可以为空
2. beanClass: 要注入的bean的class
3. supplier: 怎么得到这个 bean 的实例
4. BeanDefinitionCustomizer 这个类等同于: Consumer<BeanDefinition>，我们可以去设置 bean 的属性，比如 bd -> bd.setPrimary(true)；
```

使用示例：

```
 AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
 HLL hll = new HLL(14);
 ctx.registerBean("hll", HLL.class, () -> hll, bd -> bd.setPrimary(true));
 // 在5.2.6版本中必须调用，否则报错
 ctx.refresh();
 HLL from = ctx.getBean(HLL.class);
 System.out.println(from);
```

# 2 Wiring Beans

在 defining beans 部分学习了如何定义 bean，在OOP中组合是更推荐的方式，组合意味着一个对象要持有其他的对象。在 spring context 语境下，就是一个 bean 要引用另外一个 bean，如何完成这个引用过程就叫装配，wiring。

### 2.1 wiring的两种方式

这里其实就对应 defining beans 中的两种形式：config class 和 stereotype。其实这两种形式是同一的，都是在spring的管理之下。

#### config class

直接看示例。

```
// 唯一id
class ID {
	private static int nextId = 1;
	
	private int id;
	
	public ID() {
		id = nextId++;
	}
}

class Person {
	private ID id;
	// 省略其他
}

@Configuration
class ProjectConfig {
	
	@Bean
	public ID id() {
		return new ID();
	}
	
	// 代码点1
	@Bean 
	Person person() {
		// 这里会创建一个新的id吗？
		ID id = id();
		return new Person(id);
	}
	
	// spring 会从自己的 context 中找到类型为 ID 的 bean 把它传过来
	@Bean 
	Person person2(ID id) {
		return new Person(id);
	}
}
```

在上面的示例中值得关注的是代码点1的部分，它在方法内部调用了 `id()`，这会创建一个新的 `ID`对象吗？在 spring 控制之下的运行不会，spring 会认为你是想要引用 `id` 这个 bean，而不是要去创建一个新的对象，此时有两种情况：

1. context 中已经有这个 bean 了，直接返回 bean 的引用；
2. context 中没有这个 bean，先在 context 中创建这个 bean, 然后把 bean 的引用返回；

但是如果不是在 spring 的控制之下，那就要遵循 Java 的逻辑：每次方法调用都是一次独立的方法执行，在这里就会创建一个新的ID对象。

```java
// 实际情况下不可能这么使用
ProjectConfig config = new ProjectConfig();
// 创建一个新的 ID 对象
config.id();
// 先创建一个新的 ID 对象，然后再创建一个新的 Person 对象
config.person();
```

#### Stereotype: 使用 @Autowired 注解

Spring 支持在 3 个地方使用自动注解

##### 1. auto wired by class field(不推荐，经常出现在演示类代码中)

```
class Person {
	@Autowired
	private ID id;
}
```

这种形式不推荐，为什么呢？

1. 字段不能被定义为 `final`的
2. 很难自己去初始化这个类

todo:  Spring 在什么时候完成的字段注入？

##### 2. auto wired by constructor(推荐的方式)

```
class Person {
	// Notice: id is final
	private final ID id;
	
	@Autowired
	public Person(ID id) {
		this.id = id;
		do something else...
	}
}
```

这种方式可以克服之前说的确定，并且可以在构造方法中去做其他的事情。这种方式有两点值得注意：

1. 一个类只能有一个构造方法使用 @Autowired 注解
2. 从 Spring 4.3 开始，如果一个类只有一个构造方法，连@Autowired注解都可以省略

##### 3. Auto wired by setter (不推荐: 缺点多于优点)

```
class Person {
	// Notice: id is final
	private final ID id;
	
	@Autowired
	public void setId(ID id) {
		this.id = id;
	}
}
```

config class 和 stereotype 两种方式是可以混用的，都是 spring context 管理的 bean。

#### 2.2 同一种类型有多个 bean 是如何选择

有三种方式，优先级依次降低

##### 1. @Qualifier(bean name): 推荐，优先级最高

```
class HLL {
	private int p;
}

// bean 定义
@Primary
@Bean
public HLL hll() { ... }

@Bean
public HLL hll16() { ... }

@Bean 
public HLL hll18() { ... }

class HMH {
	private HLL hll;
	
	// 这里注入的是 hll 18
	@Autowired
	public HMH(@Qualifier("hll18") hll) {
		this.hll = hll;
	}
}
```

##### 2. @Primary： 默认选择

```
bean 定义如上

// 这里注入的是 hll, 因为它是 primary
@Bean 
HMH hmh16(HLL hll16) { ... }
```

##### 3. 参数名称、使用@Autowired的字段名称和 bean name 一致： 不推荐，很容易在对参数重命名时破坏

```
// bean 定义
@Bean
public HLL hll() { ... }

@Bean
public HLL hll16() { ... }

@Bean 
public HLL hll18() { ... }

// 这里注入的是 hll16, 没有 primary 的情况下，默认将参数同名bean注入进来
@Bean 
HMH hmh16(HLL hll16) { ... }
```



