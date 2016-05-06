# Apache Parquet

## Motivation

我们利用压缩和列式存储的优势创建了Parquet来应用到Hadoop生态的项目。

并且利用了嵌套数据结构和Google Dremel paper中的算法 [record shredding and assembly algorithm](https://github.com/Parquet/parquet-mr/wiki/The-striping-and-assembly-algorithms-from-the-Dremel-paper)来从头开始构建Parquet。

Parquet被设计成支持高效的数据压缩和编码格式，多个项目已经表明对数据应用正确的压缩和编码可以提高性能。Parquet可以按列指定压缩的格式，并且允许增加更多编码实现，这样的方法是永不过时的。


