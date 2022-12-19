---
title: "CMU 15-445 Project2 总结"  
excerpt: "地狱b+树"
classes: wide
header:
  image: /assets/images/project2/1.jpg  
share: false
---

**[CMU 15-445](https://15445.courses.cs.cmu.edu/fall2022/)**是CMU关于数据库设计和实现的一门基础课程，对全校的本科生和研究生开放。2022年这门课由数据库领域的大牛 **[Andy Pavlo](http://www.cs.cmu.edu/~pavlo/)**讲授，课程内容涵盖存储引擎，索引模型，查询优化和并发运算等多方面知识，非常适合数据库初学者将其作为数据库的第一门课程学习。

project2的目标是写一个b+树作为索引结构，难度相对project1可谓直线上升，由于b+树的结构过于复杂，光是想要写完GetValue, Insert和Remove就要花费大量时间和精力，到最后写并发逻辑的时候更是写得人脑袋痛。不过通过这个project，对b+树的各种操作有了更深入的了解，证明其本身还是有含金量的。project1共分为四个部分：

  * Task #1 - B+Tree Pages
  * Task #2 - B+Tree Data Structure (Insertion, Deletion, Point Search)
  * Task #3 - Index Iterator
  * Task #4 - Concurrent Index

## Task #1 - B+Tree Pages

这部分的任务是完善b_plus_tree_page，b_plus_tree_internal_page和b_plus_tree_leaf_page的实现，本身并不难，不过难点在于搞清楚两种page之间的区别和page本身的结构。先从b_plus_tree_page本身讲起吧。

  * page_type_ : 界定其到底是internal page还是leaf page，一开始我对此有些疑惑，因为根据教材**Database System Concepts**的说法，page可分为root page，internal page和leaf page三种。不过后来想到root page可通过parent_page_id_来判断。
  * lsn_ : log sequence number会在project4用到。
  * size_ : page内key&value组成的pairs的数量，这里我想提一下，在leaf page里，key和value的数量是相等的;而在internal page里，由于第一个节点的key不存在，key比value少一个，这造成了internal page后续实现的一些困难，总体来说，internal page的实现相较leaf page略难。
  * max_size_ : 顾名思义，page的最大大小，超出此大小的节点会分裂。
  * parent_page_id_ : 顾名思义，父节点的page_id，若为-1则说明为root page。
  * page_id_ : 自身page_id。

b_plus_tree_page本身的实现没有什么难度，值得一提的是GetMinSize()，需要根据是否为root page分类讨论。

接下来讨论b_plus_tree_internal_page，其剩下来用于存储key&value的空间为INTERNAL_PAGE_SIZE = ((PAGE_SIZE-INTERNAL_PAGE_HEADER_SIZE) / (sizeof(MappingType))。其中PAGE_SIZE为4096B，INTERNAL_PAGE_HEADER_SIZE表示上述六条header的大小，为24B。至于存储key&value的数据结构，直接用array_[1]表示(由于编译器的原因，将其改为array_[0]才通过编译，很奇怪)。这其实是一种 **[Flexible array](https://en.wikipedia.org/wiki/Flexible_array_member)**,事实上，Flexible array必须位于结构体的末尾(否则无论申请多大的内存都没法扩充了)，其实际大小由申请的内存决定。另外，internal page中value是page_id_t，和leaf page不一样。

最后是b_plus_tree_leaf_page。其相比internal page多了next_page_id_，方便写迭代器。另外leaf page的value是RID，由page_id和slot_num_(在page中的位置)构成。最后leaf page中key和value的数量是相等的。除上述三条之外，与internal page实现逻辑相差不大。

## Task #2 - B+Tree Data Structure (Insertion, Deletion, Point Search)

**Point Search**是最简单的部分，先贴一段书上的伪代码，照着它实现就行。

<figure>
    <a href="/assets/images/project2/2.jpg"><img src="/assets/images/project2/2.jpg "></a>
</figure>

需要注意的是，在查找leaf page的过程中，每从buffer pool里fetch一个page，需要先将其转换为相应种类的page(internal page或leaf page)，

```c++
BPlusTreePage *node = reinterpret_cast<BPlusTreePage *>(page->GetData());
```

而在page用完，可以丢弃的时候，需要及时unpin，否则遍历过的page会占满buffer pool。

之后是**Insertion**，书上的伪代码如下：

<figure>
    <a href="/assets/images/project2/3.jpg"><img src="/assets/images/project2/3.jpg "></a>
</figure>

<figure>
    <a href="/assets/images/project2/4.jpg"><img src="/assets/images/project2/4.jpg "></a>
</figure>

Insertion的难点在于split部分，当每个leaf page在插入之后达到最大容量max_size_的时候，就需要分裂。当第一次发生分裂的时候，总是leaf page分裂，此时需要创建一个新leaf page，称为sibling page。这个sibling page在原来page的右侧，将原来page中一半的key&value转移到sibling page后，page和sibling page的关系就构建完成了。当然，最后还需要将其与parent page建立联系，首先在init sibling page时把parent_page_id_设为和page一样，其次在parent page的array_中对应的位置插入代表sibling page的key&value。

当然，若parent page在插入后满负荷，说明parent page本身也需要分裂。由于parent page是internal page，其分裂的逻辑和leaf page有所不同。在将原来page中一半的key&value转移到sibling page时，需要关注key&value代表的chlid page，应将它们的parent page设为sibling page。

**Deletion**是较难的部分，伪代码如下：

<figure>
    <a href="/assets/images/project2/5.jpg"><img src="/assets/images/project2/5.jpg "></a>
</figure>

整个deletion的关键在于Coalesce和Redistribute上面。当对leaf page进行删除后，若此leaf page的大小小于minsize，就需要Coalesce或者Redistribute。和insertion中的流程类似，当leaf page大小过小时需要先找到sibling page，不过这次sibling page的理想情况实在page的左边。当整个page和sibling page的大小加起来小于sibling page的max_size_时，意味着page的所有节点可以放入sibling page，所以这时选择Coalesce。leaf page的Coalesce流程和spilt类似，将page的节点转移到sibling page后再删除parent page中代表page的节点即可。而Redistribute代表要向sibling page借一个节点，不管sibling page是在page的左侧还是右侧，总会有一个page的首节点发生变化，因此要注意修改parent page中对应的节点。

和insertion类似，若parent page的大小过小，则也需要Coalesce或Redistribute，对internal page的Coalesce不复杂，注意将page的所有chlid page修改到sibling page名下即可。而Redistribute有些麻烦，这里我画了个示意图(示意图中sibling page在page的左侧，对于在右侧的情况类似处理)：

<figure>
    <a href="/assets/images/project2/6.jpg"><img src="/assets/images/project2/6.jpg "></a>
</figure>

当需要把sibling page的最后一个节点移动到page的首位时，由于internal page的第一个节点的key是无效的，所以当page中的节点整体右移一个单位之前，要先把page的第一个节点变为有效节点，方法是把parent page中对应的key转移到page的第一个key&value节点上，之后再按部就班转移节点和更新child page中父节点的page id即可。


## Task #3 - Index Iterator

这部分没有什么难度，构建iterator时设置三个成员：

  * BufferPoolManager *buffer_pool_manager_
  * Page *page
  * int idx

buffer_pool_manager_用于fetch page，后两者可以确定一个key&value在B+树中的位置。

## Task #4 - Concurrent Index

在讲Concurrent Index之前，先看一下**crabbing protocol**，在 **[课程slides](https://15445.courses.cs.cmu.edu/fall2022/slides/09-indexconcurrency.pdf)**上已有了很好的示范。简而言之，对于Insertion, Deletion, Point Search，向下寻找目标page时，需要对路径上的所有page依次上锁，至于解锁的条件，依赖于其chlid page的大小:

  * 对于Insertion，chlid page的size_ < max_size_ - 1时，chlid page必定不可分裂，因此chlid page之上的所有page均可解锁
  * 对于Deletion， chlid page的size > min_size_ 时，chlid page必定不会发生Coalesce/Redistribute，故chlid page之上的所有page均可解锁
  * 对于Point Search，由于chlid page不发生改变，故获取chlid page的锁后即可对chlid page之上的所有page解锁

对于Concurrent Index的实现，依赖于Transaction类，每次Insertion/Deletion/Point Search的过程由一个Transaction实体管理。其中对本次project较为重要的成员为：

```c++
/** Concurrent index: the pages that were latched during index operation. */
std::shared_ptr<std::deque<Page *>> page_set_;
/** Concurrent index: the page IDs that were deleted during index operation.*/
std::shared_ptr<std::unordered_set<page_id_t>> deleted_page_set_;
```

前者负责记录在Insertion/Deletion/Point Search的过程中被锁的page，后者负责记录在Deletion过程中需要被删除的page。使用方法是：前者记录在搜索路径中的page，并对Insertion中新增的page和Deletion中的sibling page进行保护，同时根据上述规则及时解锁；后者仅在Deletion中使用，并在Deletion的末尾删除其中的page。

另外，对于根节点为leaf page的情况(也即仅有一个节点)，由于无法通过chlid page判断根节点的安全性，需要单独对根节点上锁，等到能够判断根节点安全时再解锁。

最后，对于Index Iterator的并发控制，比较省事的方案是在实现operator++()时，每新到一个page就对新page加锁，对旧page解锁。


