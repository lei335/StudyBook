# CRUSH算法

crush算法是按照crush map进行运算的。

crush模块的源码路径为src/crush，其中：

> - **crush.h 和 crush.c** : *crush map* 的基本数据结构
> - **build.h 和 build.c** : 实现了如何构造 *crush_map* 数据结构
> - **CrushCompiler.h 和 CrushCompiler.cc** : 解析 *crush_map* 的词法和语义，相当于翻译crush map 文件
> - **CrushWarpper** : 是CRUSH核心实现的封装
> - **mapper.h 和 mapper.c** : CRUSH 算法的核心实现

crush map由5部分组成：tunable参数，device，type，bucket，rule，在代码中的结构：

![img](https://dovefi-1256247019.cos.ap-guangzhou.myqcloud.com/ceph/crush%20map%20struct.jpg)