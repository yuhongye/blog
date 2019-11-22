## 0. 文件结构
file contains multi row group
   row group contains multi column chunk, but only one chunk per column
      column chunk contains multi data page

## 1. metadata
`ParquetMetadata`完整的表示一个parquet file footer中的metadata, 它包含`FileMetaData`和`List<BlockMetaData>`。

`FileMetaData` 表示文件级别的metadata, 它可能是读取了多个parquet file后merge得到的信息，merge由`GlobalMetadata`来完成
* createBy: 由谁创建的，有可能读去了多个版本的parquet file，因此它可能是一个Set.toString
* keyvalueMetadata: Map<String, String> 文件级别的配置信息
* schema: 文件schema

`BlockMetadata` 表示一个row group级别的metadata, 它包含了一个row group中所有列的元信息:
* rowCount: row group中包含行的条数
* totalByteSize: 从目前来看，应该是压缩后的数据大小
* path: 包含该row group的file path
* columns: List<ColumnChunkMetaData>， 列在该row group中的column chunk metadata

`ColumnChunkMetaData` 一列在row group中的chunk信息
* codec: 压缩算法的名字
* path: ColumnPath, parquet 特有的，还没整明白是个啥
* type: PrimitiveType 从名字看时该列是何种基础类型
* encodings: Set<Encoding> 为什么是Set?
* first data page offset: 难道所有的data page都是紧挨的，并且大小固定？如何读取data page
* dictionary page offset: 如果使用了dictionary编码的话，会有该值
* value count: 值的个数
* toatalSize: 看起来是压缩后的数据大小
* totalUncompressedSize: 未压缩的大小
* Statistic: 统计信息

`ColumnChunkMetaData`中存储了一些信息，比如offset和count之类，它有两个子类，分别是`IntColumnChunkMetaData`和`LongColumnChunkMetaData`，如果数据可以放在int中则使用`IntColumnChunkMetaData`，否则使用long类型，据说是为了节省内存？有必要吗?