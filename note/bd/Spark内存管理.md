## 1. Spark 1.6及之后的内存模型
```java
+---------------+
|               | 缓存数据
|    Storage    | spark.memory.storageFraction
|  ^            | Storage不可以驱逐Execution的内存
+--|------------+ spark.memory.fraction
|  V            | Execution可以驱动Storage，最多驱逐至Storage只剩R
|   Execution   | shuffle, agg
|               |
+---------------+
|   Other       | 用户自定义数据结构，spark元数据，防止OOM
+---------------+
|Reserved(300MB)| 预留数据, 防止OOM
+---------------+
``
Storage和Execution共享一块统一的内存空间，并且可以互相扩展。假设java heap的总大小是H:

* Storage和Execution共享: M = (H-300) * spark.memory.fraction
    - Storage: S = M * spark.memory.storageFraction
    - Execution: E = M - S
* Other: (H-300) - M

