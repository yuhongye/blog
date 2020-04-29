## 1. column write的逻辑
在parquet-column/example中把data分成`Group`和`Primitive`，这两类对象都借助`RecordConsumer`来对record进行序列化，也就是由row转成column。真正调用的逻辑在`GroupWriter`类中, 它只对外暴露了一个借口`write(Group group)`，类的代码如下：
```java
public class GroupWriter {
    private final RecordConsumer recordConsumer;
    private final GroupType schema;

    /**
     * 这个地方group应该必须要跟schema对应起来
     */
    public void write(Group group) {
        recordConsumer.startMessage();
        writeGroup(group, schema);
        recordConsumer.endMessage();
    }

    /**
     * 遍历type, 如果是primitive type， 则首先去group里面获取它对应的Primitive, 比如IntValue的实例，然后调用IntValue.writeValue(RecordConsumer),
     *  其实这些primitive type对应的writeValue()都是直接调用RecordConsumer.addXXX(value)
     * 如果是group type，在仍然需要递归的遍历 
     */
    private void writeGroup(Group group, GroupType type) {
        for (each field in type) {
            recordConsumer.startField(fieldName, fieldIndex);
            if (filed is primitive type) {
                group.writeValue(field, fieldIndex, recordConsumer);
            }  else {
                recordConsumer.startGroup();
                writeGroup(group.getGroup(field, fieldIndex), fieldType.asGroupTye());
                recordConsumer.endGroup();
            }
            recordConsumer.endField(fieldName, fieldIndex);
        }
    }
}

```

`ValueWriter`接口提供了各种类型的编码方法，比如Parquet中提供了`BitPackingValueWriter`, `DeltaBinaryPackingVAluesWriteForLong`等。

`ColumnWriter`类提供了`write(value, repetition level, definition level)`的方法， 这是对一列数据进行保存的地方，在parquet中，一列数据包括：value， repetition level， definition level, 因此它的子类一般要有4个属性：

* RunLengthBitPackingHybridEndoder repetitionLevelColumn, 用来保存repetition level
* RunLengthBitPackingHybridEncoder definitionLevelColumn, 用来保存definition level
* ValueWriter dataColumn，用来写真正的列的数据
* PageWriter pageWriter，当写入的数据达到一定量时，也就是足够一个page时，把当前写一个page出来

`PageWriter`接口，就是用来执行实际的page write，比如在test目录下提供了一个`MemPageWriter`，就是把page保存在List中。

### 方法调用链
```java
+---------------+ create  +---------------+ create  +---------------+  
|ColumnIOFactory| ------> |MessageColumnIO| ------> | RecordConsumer|
+---------------+         +---------------+         +---------------+

+-----------------------------------------------------------------------+
|GroupWriter.write(Group):                                              |
|  RecordConsumer.startMessage();                                       |
|  for each field                                                       |
|      RecordConsumer.startField()                                      |
|      if primitive type then Group.writeValue(field, RecordConsumer);  |
|      if group type     then recursive group type, write its children  |
|      RecordConsumer.endField()                                        |
|  RecordConsumer.endMessage();                                         |
+-----------------------------------------------------------------------+

       Group                                  Group                        Primitive  
+---------------------------------+      +----------------+       +---------------------------+
|writeValue(field, RecordConsumer)| ---> | get field value|  -->  | writeValue(RecordConsumer)|
+--------------------- -----------+      +----------------+       +---------------------------+
                                                                              |
           +------------------------------------------------------------------+
           |
           V
+-----------------------------+      +---------------------------------------------------+
| RecordConsumer.addXXX(value)|  --> | ColumnWriter.write(value, repeatition, definition)|
+-----------------------------+      +---------------------------------------------------+
```

# 怎么计算definition level和repetition level