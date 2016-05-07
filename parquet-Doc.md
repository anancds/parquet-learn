# Apache Parquet

## Motivation

我们利用压缩和列式存储的优势创建了Parquet来应用到Hadoop生态的项目。

并且利用了嵌套数据结构和Google Dremel paper中的算法 [record shredding and assembly algorithm](https://github.com/Parquet/parquet-mr/wiki/The-striping-and-assembly-algorithms-from-the-Dremel-paper)来从头开始构建Parquet。

Parquet被设计成支持高效的数据压缩和编码格式，多个项目已经表明对数据应用正确的压缩和编码可以提高性能。Parquet可以按列指定压缩的格式，并且允许增加更多编码实现，这样的方法是永不过时的。


Parquet可以被任何人使用，Hadoop生态有丰富的处理框架，但我们并不会厚此薄彼。我们相信一个高效的，底层有良好实现的列式存储必须对所有的框架都有效，并且不用建立大量的复杂的依赖

##Modules

**parquet-format**工程包含了格式说明和元数据的Thrift定义，这些元数据用来正确的读取Parquet文件。

**Parquet-mr**工程包含很多子模块，这些子模块实现了一些核心组件，比如读取和写入嵌套和面向列式的数据流，然后映射到Parquet的格式，比如提供了Hadoop的输入输出格式，比如Pig加载器，还有别的基于java用于和Parquet交互的工具类。

**Parquet-compatibility**工程包含了兼容性测试，用来核实不同语言之间的实现能否相互读取和写入彼此的文件。

##Building

获取Java的资源可以用如下的命令。目前稳定的版本可以从maven中央库中获取。

>mvn package

获取c++的资源可以通过**make**生成。

Thrift文件可以生成到任何支持Thrift的语言。

##Releasing
[How to Release](https://parquet.apache.org/documentation/how-to-release/)

##Glossary

Block(hdfs block):就是hdfs中的一个块，也就意味着描述Parquet格式的文件是不能改变的。parquet文件这样的设计可以很好的跑在hdfs上。

File：一个hdfs文件，必须包含Parquet文件的元数据，但是不一定包含数据。

Row Group：逻辑上水平分区的数据划分成行，每一个行组保证没有物理结构。对于数据集中每一列数据，一个行组包含一个列块。

Column Chunk：对于特殊的一列中的一块数据，这些数据存在于特殊的行组中并且保证在文件中是连续的。

Page： 列块划分为多个页，一页是最小的压缩和编码单位，在概念上已经不可分割。在同一个列块中可以有不同压缩编码类型的页

按分层来说，一个文件包含一个或者多个行组，一个行组内的每一列包含一个列块，一个列块包含一个或者多个页。

## Unit of Parallelization(并行度)
* MapReduce - File/Row Group
* IO - Column chunk
* Encoding/Compression - Page

##File format

Parquet文件和Thrift的定义需要一起来看才能理解这些格式

	4-byte magic number "PAR1"
	<Column 1 Chunk 1 + Column Metadata>
	<Column 2 Chunk 1 + Column Metadata>
	...
    <Column N Chunk 1 + Column Metadata>
    <Column 1 Chunk 2 + Column Metadata>
    <Column 2 Chunk 2 + Column Metadata>
    ...
    <Column N Chunk 2 + Column Metadata>
    ...
    <Column 1 Chunk M + Column Metadata>
    <Column 2 Chunk M + Column Metadata>
    ...
    <Column N Chunk M + Column Metadata>
    File Metadata
    4-byte length in bytes of file metadata
    4-byte magic number "PAR1"

在上面的例子中，一个表包含了N列，并且分割成M行组(也可以说每个行组包含了每个列的一部分数据，这部分数据就是列块)。Parquet文件的元数据中包含所有的列元数据起始位置的信息。关于元数据中的一些细节可以参考Parquet的Thrift文件。

当数据允许单方面写时，元数据会被写入。

读取者希望第一次读文件的元数据时就读到他们感兴趣的列块信息，然后列块必须被顺序的读取。

![](./resource/FileLayout.gif)

* 以下是个人理解(不一样的视角)：

一个文件分成了很多列(column),比如Columna，Columnb，Columnc，一个列又分成了很多ColumnChunck,这些ColumnaChunck0，ColumnbChunck0，ColumncChunck0组成了ColumnGroup0，所以，也可以说是一个文件分成了很多ColumnGroup，每个ColumnGroup包含Columna，Columnb，Columnc...中的一部分数据，称之为ColumnChuck，每个ColumnChunck有分为很多Page

	| Columna       |Columnb        | Columnc      |... |
	| ------------- |-------------  | -----        |---:|
	| ColumnaChunck0| ColumnbChunck0|ColumncChunck0|... |
	| ColumnaChunck1| ColumnbChunck1|ColumncChunck1|... |
	| ColumnaChunck2| ColumnbChunck2|ColumncChunck2|... |
	|...			|...  			|...		   |... |

## Metadata

有三种类型的元数据：文件元数据，列元数据，页头元数据。所有的Thrift结构都是用TCompactProtocol协议序列化。

![](./resource/FileFormat.gif)

## Types

文件格式支持的类型是越小越好，因为类型会影响磁盘存储。比如存储格式明确不支持16位的整型，因为32位的整型有更高效的编码。这样的设计降低了实现对文件格式读取和写入的复杂度。

类型有：BOOLEAN：1 bit boolean、INT32：32位有符号整型、INT64：64位有符号整型、INT96:96位有符号整型、FLOAT：IEEE 32-bit浮点型、DOUBLE：IEEE 64-bit 位浮点型、BYTE_ARRAY：arbitrarily long byte arrays

###Logic Types

逻辑类型通过指定如何解释原始类型的方式扩展了Parquet可以存储的类型。这种方式保证了原始类型的数据比较少并且复用了Parquet的高效的编码方式。比如Strings用UTF8注解的byte arrays存储，这些注解定义了今后如何解码和解释数据。注解以**ConvertedType**的形式存储在文件的元数据中，说明文档在[LogicalTypes.md](https://github.com/Parquet/parquet-format/blob/master/LogicalTypes.md "LogicalTypes.md")。

##Nested Encoding

为了对嵌套的列编码，Parquet采用了definition and repetition标准的Dremel编码。Definition 层指定在列这个层次上有多少个可选的字段，Repetition层定义了字段上有多少个重复的值。definition and repetition的最大值可以通过scheme文件直接计算。

##Nulls

空值在definition层编码，数据中的空值并不会编码。比如在一个没有嵌套的schema文件中，一列有1000个空值会在definition层编码成(0, 1000times)。

##Data Pages

对于数据页，在页头后，有三种信息会被连续编码。如下：

*  definition levels data
*  repetition levels data
*  encoded values
文件头的大小就是由这三个指定的。

根据schema文件的定义，数据页的数据是需要的，definition and repetition层是可选的。如果列不是嵌套的，那么我们不会编码repetition层。因为数据是需要的，所以definition层会被跳过。

参考：[Encodings.md](https://github.com/Parquet/parquet-format/blob/master/Encodings.md "Encoding.md")

## Column chuncks

列块由连续的页组成。页共享同一个头，并且读取者可以跳过他们不感兴趣的页。页中的数据接着投存储，并且可以压缩和编码。压缩和编码格式在页的元数据中指定。

##Checksumming

数据页可以单独校验，这样就可以关闭HDFS文件层的校验，可以更好的支持单个行的查找。

##Error recovery

如果文件的元数据毁坏，那么文件就会丢失。如果列的元数据毁坏，那么列块就丢失。如果页头元数据毁坏，那么列块中所有其他的页都是丢失。如果页中的数据毁坏了，那么整个页就会丢失。所以如果行组比较小的话，那么Parquet文件就会更有弹性。

潜在的扩展：如果是小的行组，那么大问题就是最后怎么放置这些文件的元数据。如果在写文件元数据时发生错误，所有写入的数据都会不可读。这个可以通过每隔第n行行组就写一次数据解决。

每一个文件元数据都会累积并且会包含所有的行组，用rc或者avro文件异步且策略性的合并这些文件，那么读取者就可以恢复部分写入的文件。

##Separating metadata and column data

文件格式显然设计成把元数据从数据中隔离出来。这样就可以允许列分散到不同的文件中，并且有一个单独的元数据文件指向多个Parquet文件。

##Configurations

* Row group size: 越大的行组就允许越大的列块，这样就可以提供高顺序I/O,更大行组还可以提供写路径时更多的缓冲。我们推荐很大的行组(512MB-1GB)，因为一整个行组都有可能被读取，我们想要完整的匹配HDFS的block。所以HDFS的block也需要被设置的大一点。一个比较优读取配置是：1GB行组，1GB HDFS block，每个HDFS文件有1个HDFS block。
* Data page size:  数据页被认为是不可分割的，所以小点的数据页允许更细粒度的读取。页太大会引发更少的空间损耗(更少的页头)和潜在的解析损耗(处理头)。

注意：对于顺序扫描，并不希望每次都读取一个页，这不是I/O块，我们推荐的页大小是8KB。

##Extensibility

文件格式有很多地方用来兼容性扩展：

* File Version:文件元数据包含一个版本。
* Encoding: 编码用枚举值指定，并且今后可以扩展。
* Page types: 可以添加额外的页类型并且可以安全的跳过。