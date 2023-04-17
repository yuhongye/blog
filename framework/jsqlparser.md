### Node

`Node` 只有一个子类 `SimpleNode`, `SimpleNode` 有 `parent`, `children`, `value`等属性，这样也能构成一个树，但是没有对应的对象结构。

现在有一个地方没看懂，`ASTNodeAccess`接口，它提供了 `SimpleNode getASTNode()`和`void setASTNode(SimpleNode)`，它只有一个直接实现类`ASTNodeAccessImpl`, 它包含 `SimpleNode node`属性，没有提供额外的功能。所有的`Expression`的子类都是`ASTNodeAccessImpl`的子类。

### Expression

`Expression`的类层级(节选)

![Expression类图](/Users/caoxiaoyong/Documents/blog/framework/images/JSqlParser_Expression.png)

下面就关心的几类来介绍:

* 常量: `LongValue`, `StringValue`,`NullValue`等
*  数据表中的列: `Column`，它有两个属性：`Table table`和 `String columnName`
*  二元表达式: `AndExpression`和四则运算是其中的典型
* 函数调用: `Function`, 目前注意到它的几个属性：
  * `nameparts: List<String>`， 比如`json.fromJson(desc, "name")`，函数名就是有: "json", "fromJson"构成
  * `parameters: ExpressionList`，比如 `fromJson(desc, "name")`, 它的参数是两个`Expression`: `Column(desc)`和常量`name`
* `case when` 表达式
* `()`用来改变优先级: `Parenthesis`

下面举一些例子类理解它: `CCJSqlParserUtil.parseExpression(expression)`

1. `"parseExpression(\"a\"")`等同于 `parseExpression("a")`会被当成`Column(a)`
2. `parseExpression("'a'") 会转成`StringValue(a)`
3. `(count(distinct id) + if(a = 1, 1, 0)) + md5(json.fromJson(desc, 'name'))`它会被解析成下面这棵树![parser_expression](/Users/caoxiaoyong/Documents/blog/framework/images/jsqlparser_parser_expression.png)

