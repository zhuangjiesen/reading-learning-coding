# Eclipse MAT工具使用

官网文档
http://help.eclipse.org/oxygen/index.jsp

1. Shallow vs. Retained Heap
Shallow heap 是一个对象消费的内存大小，一个对象引用需要 32 或者 64 位内存(取决于操作系统架构) ， Integer 4个字节 , Long 8 个字节

Retained heap 是 一个对象存活时，占用的内存总和 (包括对象中的引用)


打开hprof 文件进行分析时
1. Histogram: Lists number of instances per class
2. Dominator Tree: List the biggest objects and what they keep alive.
3. Top Consumers: Print the most expensive objects grouped by class and by package.
4. Duplicate Classes: Detect classes loaded by multiple class loaders.


5.Thread detail 分析


