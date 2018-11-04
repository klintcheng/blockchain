
# bitcoin源码分析

[btcd][2]是用golang编写的bitcoin全节点客户端。分析源码的过程类似于树的两种遍历算法，深度优先或者广度优先。如果按深度优先去看，会陷入非常复杂的细节方法中，前期会非常难懂，坚持不下去。因此，我会先以广度优先为主，理出一条条线索，按着线索把各个组件的功能功用有个大致的了解。所以在一些章节中我不会把所有的细节方法写上来，只要知道它的意思就行了。重点不是看代码说明，而是代码本身。


## 目录

1. [源码结构](btcd_1.md)
2. [系统启动过程](btcd_2.md)
3. [blockchain](btcd_3.md)
4. []

[1]:<https://github.com/bitcoinbook/bitcoinbook>
[2]:<https://github.com/btcsuite/btcd>