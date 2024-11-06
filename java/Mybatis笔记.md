### 0 简介

1. 与其他ORM框架不同，Mybatis没有将Java对象与数据库表关联起来，而是将Java方法与SQL语句关联。
2. MyaBatis提供了一个映射引擎，声明式的将SQL语句的执行结果与对象树映射起来。
3. MyBatis是一个结果映射框架。

### 1 注意点

1. 每个数据库环境应该就一个 `SqlSessionFactory`实例，它应该是一个单例模式
2. 每次使用完 `SqlSession` 之后要记得 `close()`还给 `connection pool`，否则会发生连接泄露。
3. 每个线程应该使用自己的`SqlSession` 实例，`SqlSession` 不是线程安全的，不能被共享。从 Web 应用程序角度上看， `SqlSession` 应该作用在 `request` 级别上。