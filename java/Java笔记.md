### String

##### String interning

在计算机科学中，string interning是一种用来优化手段，对于每个string只保存一个副本，但是要求string必须得是immutable。在Java中，使用string pool来保存intern string，字符字面量自动放到string pool中，非字符字面量则必须要调用`intern()`方法：它会使用`equals(Object)`去跟string pool中的做比较，如果遇到相等的则返回pool中已有的对象，否则把该对象添加到pool中。

什么叫字符串字面量，大体上理解是：可以在编译器确定的常量。举几个例子:

```java
String hello = "Hello"; // 是字面量
String s1 = "Hel" + "lo"; // 是字面量
String lo = "lo";
String s2 = "Hel" + lo; // 不是字面量，必须要到运行时才能生成
// 由于有了上面的字面量知识，因此下面的语句
boolean isFalse = ("hel" + lo) == ("hel" + lo);
boolean isFalse = ("hel" + lo) == ("hel" + lo).intern();
boolean isTrue = ("hel" + lo).intern == ("hel" + lo).intern();

```

##### String Deduplication in G1

接着上条说，系统中存在着大量的String对象，它们底层的数据是一样的，因此造成了巨大的浪费.G1的开发者就希望能把这些相等的对象给去重，大体上就是把String对象底层依赖的`char[]`在jvm层面给替换成同一个，这样虽然String对象没有去重，但是占用内存的大头也就是String的内容是去重了的.