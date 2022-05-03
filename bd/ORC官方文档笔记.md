# 1 介绍

ORC(Optimized Row Columnar) 是一种自描述的强类型列存文件格式。它的设计目标是为了流式的读取大量数据而设计，也可以兼容快速点查的场景。

谓词下推可以减少数据加载：

1. index可以快速定位到 stripe 级别
2. row index 可以帮助定位 1万行的级别(ORC会对每1万行存储index和最大/最小值)

ORC 文件分成 stripes，每个 stripes 之间独立，因此可以方便的进行 stripe 级别的并行。stripe 内部的每个列单独存储。因此整个ORC文件可以说是行列混合存储。