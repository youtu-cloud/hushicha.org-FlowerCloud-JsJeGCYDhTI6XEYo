
目录* [Overview](https://github.com)
* [Nested Loop Join](https://github.com)
	+ [Naïve](https://github.com)
	+ [Block](https://github.com)
	+ [Index](https://github.com)
* [Sort\-Merge Join](https://github.com):[westworld加速](https://tianchuang88.com)
* [Hash Join](https://github.com)
	+ [Simple Hash Join](https://github.com)
	+ [Partition Hash Join](https://github.com)
* [总结](https://github.com)

## Overview


输出形式：早物化与晚物化（OLAP一般都是晚物化）


代价分析：一般用IO次数计算（最终结果可能落盘，也可能不落盘，所以我们只计算输出结果之前的IO次数）。


*Join*左边称为外表（Outer Table），右边称为内表（Inner Join），外表一般是小表。


![](https://my-pic.miaops.sbs/2024/11/image-20241115130739507.png)
## Nested Loop Join


### Naïve


前提：缓冲区大小为3，一个外表输入，一个内表输入，一个输出。


基本思想：双重循环，对每一个元组（Tuple）进行配对，读取S表m次。


Cost：M\+(m∗N)M\+(m∗N)



![image](https://my-pic.miaops.sbs/2024/11/image-20241115131024551.png)
![image](https://my-pic.miaops.sbs/2024/11/image-20241115131045387.png)

### Block


前提：缓冲区大小为3，一个外表输入，一个内表输入，一个输出。


基本思想：双重循环，对每一个块（Block，同页Page）内进行配对，所以读取S表M次。


Cost：M\+(M∗N)M\+(M∗N)


![image-20241115131544624](https://my-pic.miaops.sbs/2024/11/image-20241115131544624.png)


如果缓冲区容量为B，即可以容纳B个块（页），B\-2个块用于外表输入，一个块用于内表输入，一个块用于输出。


Cost：M\+(⌈M/(B−2)⌉∗N)M\+(⌈M/(B−2)⌉∗N)


### Index


前提：缓冲区大小为3，一个外表输入，一个内表输入，一个输出。


基本思想：如果外部表有索引，那么内层循环无需遍历，查询索引即可。


Cost：M\+(m∗C)M\+(m∗C)


![image-20241115132426075](https://my-pic.miaops.sbs/2024/11/image-20241115132426075.png)


## Sort\-Merge Join


基本思想：排序后的序列更容易找到匹配项。


分为两个步骤：


1. 排序：用任意排序方式，将R和S排序。
2. 合并：移动两个指针寻找匹配项，过程中可能需要回退指针。



> 这两个步骤和上一节提到的[外部归并排序](https://github.com)思想相同，但不是同一个东西。


SortCost(R)：2M∗(1\+⌈logB−1⌈M/B⌉⌉)2M∗(1\+⌈logB−1⌈M/B⌉⌉)


SortCost(S)：2N∗(1\+⌈logB−1⌈N/B⌉⌉)2N∗(1\+⌈logB−1⌈N/B⌉⌉)


MergeCost：M\+NM\+N


Total Cost：Sort \+ Merge



> 当R中存的是相同元素，且S中也是时，指针需要一直回退，Sort\-Merge Join退化为Nest Loop Join。


![image-20241115140700767](https://my-pic.miaops.sbs/2024/11/image-20241115140700767.png)


![image-20241115141209698](https://my-pic.miaops.sbs/2024/11/image-20241115141209698.png)


## Hash Join


### Simple Hash Join


基本思想：匹配项会被映射到同一个哈希桶。


分为两步骤：


1. 构建哈希表：对R表采用哈希函数h1' role\="presentation"\>h1h1进行哈希，得到对应的哈希桶位置，然后在哈希桶中寻找匹配项。


![image-20241115142931397](https://my-pic.miaops.sbs/2024/11/image-20241115142931397.png)


优化措施：布隆过滤器。


创建哈希表时顺带构建布隆过滤器，探测阶段先走布隆过滤器再走哈希桶。



![image](https://my-pic.miaops.sbs/2024/11/image-20241115143237351.png)
![image](https://my-pic.miaops.sbs/2024/11/image-20241115143252507.png)


> 存在的问题i：该算法需要保证哈希表能存在内存中，如果哈希表太大导致无法存到内存中，需要不断地换入换出，影响效率。但不幸的是，大部分情况下，我们都不能保证内存能完全存下哈希表。


### Partition Hash Join


基本思想：把两个表分别用同一个哈希函数哈希，相同哈希桶之间进行配对，如果哈希桶都存不下，就再哈希一次，直到能存下为止。


![image-20241115144711876](https://my-pic.miaops.sbs/2024/11/image-20241115144711876.png)


读取对应的哈希桶到内存中配对即可。


Partition Cost：2(M\+N)2(M\+N) 【读取数据\+哈希桶落盘（哈希空间复杂度为O(n)O(n)）】


Probe Cost：M\+NM\+N


Total Cost：3(M\+N)3(M\+N)


## 总结




| Algorithm | IO Cost | Example |
| --- | --- | --- |
| Naïve Nested Loop Join | M \+ (m \* N) | 1\.3 hours |
| Block Nested Loop Join | M \+ (⌈M / (B\-2\)⌉ \* N) | 0\.55 seconds |
| Index Nested Loop Join | M \+ (m \* C) | Variable |
| Sort\-Merge Join | M \+ N \+ sort cost | 0\.75 seconds |
| Hash Join | 3 \* (M \+ N) | 0\.45 seconds |


结论：选择Partition Hash Join，出现下述情况时使用Sort\-Merge Join：


* 数据偏斜严重：Hash Join退化为Sort\-Merge Join
* 数据本身需要被排序：此时Sort\-Merge Join只需要额外付出 M\+NM\+N 即可实现Join


一般数据库中，Hash Join和Sort\-Merge Join都会实现。


