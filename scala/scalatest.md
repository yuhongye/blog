## 1 test style

`scalatest` 支持多种风格的写法，推荐使用 `FlatSpec` 用于单元测试和集成测试。

### 1.1 FlatSpec

它的风格如下：

```
  /**
   * 一个主语: "An empty Set"
   * 一个动词: should, must, can
   * 剩下的语句: "have size 0"
   * in { test body }
   */
  "An empty Set" should "have size 0" in {
    assert(Set.empty.size == 0)
  }

  /**
   * 如果是相同的主语, 可以使用 it 代替
   */
  it should "produce NoSuchElementException when head is invoked" in {
    assertThrows[NoSuchElementException] {
      Set.empty.head
    }
  }
```

## 2 Assertions

Scalatest 在 `Assertions` 里定义了如下很多断言类型，并且 `Suite` 继承了 `Assertions`, `Suite` 是所有测试的基类。 提供了如下的断言类型

- `assertResult`: 更高级的形式，很明确的展示出了哪些是 `expected` 哪些是 `actual`

  ```scala
  val a = 5
  val b = 2
  assertResult(3) {
    a - b
  }
  ```

- `assume` : 如果条件不满足，会取消后面的测试

  ```scala
  val a = 4
  assume(a == 3, "a is not equals 3, skip")
  assert(a == 3) // 不会执行
  ```

- `fail` : 会抛出 `TestFailedException`

- `cancel` ：会抛出 `TestCanceledException`

- `succeed` to make a test succeed unconditionally;

- `assertThrow`s[Exception]: 必须要抛出指定的异常

  ```scala
  assertThrows[TestFailedException] {
        assert(false)
  }
  
  // 下面的测试会失败，因为没有抛出期待的异常
  // org.scalatest.exceptions.TestFailedException: Expected exception java.lang.IndexOutOfBoundsException to be thrown, but no exception was thrown
  assertThrows[IndexOutOfBoundsException] {
        Array(1, 2)(1)
  }
  
  ```

- `intercept` : 把异常捕获到这个变量里

  ```scala
  val array = Array(1)
  val caught = {
    intercept[IndexOutOfBoundsException] { // Result type: IndexOutOfBoundsException
      array(1)
    }
  }
  assertResult("1")(caught.getMessage)
  ```

- `assertDoesNotCompile` to ensure a bit of code does not compile;

- `assertCompiles` to ensure a bit of code does compile;

- `assertTypeError` to ensure a bit of code does not compile because of a type (not parse) error;

- `withClue` to add more information about a failure.

## 3 给测试添加 `tag`

可以测试用例添加 `tag` 来做归类，在运行测试的时候可以进行 `tag` 过滤

###### 1 添加预定义标签

添加 `ignore` 会忽略这个用例:

```scala
ignore should "throw NoSuchElementException if an empty stack is popped" in {
  val emptyStack = new java.util.Stack[String]
  intercept[NoSuchElementException] {
    emptyStack.pop()
  }
}
```

###### 6 自定义标签

自定义标签分两步：

1. 定义: 继承 ` `org.scalatest.Tag` 
2. 使用 `taggedAs` 添加

```java
// 自对应标签
object DbTest extends Tag("com.mycompany.tags.DbTest")

it must "subtract correctly" taggedAs(Slow, DbTest) in {
	val diff = 4 - 1
	assert(diff === 3)
}
```

