---
title: "CMU 15-445 Project1 总结"  
excerpt: "CMU手把手教你写一个存储引擎"
classes: wide
header:
  image: /assets/images/15445pro1.jpg
share: false
font-size: 2px
---

**[CMU 15-445](https://15445.courses.cs.cmu.edu/fall2022/)**是CMU关于数据库设计和实现的一门基础课程，对全校的本科生和研究生开放。2022年这门课由数据库领域的大牛 **[Andy Pavlo](http://www.cs.cmu.edu/~pavlo/)**讲授，课程内容涵盖存储引擎，索引模型，查询优化和并发运算等多方面知识，非常适合数据库初学者将其作为数据库的第一门课程学习。

**[project1](https://15445.courses.cs.cmu.edu/fall2022/project1/)**是15445关于存储引擎的一个lab，由三个小任务组成：

  * Task #1 - Extendible Hash Table
  * Task #2 - LRU-K Replacement Policy
  * Task #3 - Buffer Pool Manager Instance

## Task #1 - Extendible Hash Table

参考**[维基百科](https://en.wikipedia.org/wiki/Extendible_hashing)**，Extendible Hash Table的优点在于查询效率高，相对于普通哈希表的
链表法或再哈希法，可扩展哈希表的特点是哈希索引(Directories)直接对应哈希桶(Buckets)，而所有的数据都直接存储于哈希桶内，无需多余的查询步骤。
相应的，可扩展哈希表实现方法相对复杂，其插入和删除较有难度。

所幸project1中的可扩展哈希表并不需要实现哈希索引的缩减，降低了难度，因此本实验的难点在于插入步骤。关于可扩展哈希表的原理，可以看
**[资料](https://www.geeksforgeeks.org/extendible-hashing-dynamic-approach-to-dbms/)**。相对于普通的哈希表，需要关注的变量有Global Depth(决定索引数字的长度，同时决定索引数组的大小)，Local Depth(决定指向哈希桶指针的数量，若与Global Depth相等，则数量为一，否则为2 ^ (Global Depth - Local Depth))。

本实验中的难点在于插入，其中又可分为三种情况：

  * 新元素对应的桶未满，可以接收新元素。这种情况较为简单，直接对相应的桶作插入即可。
  * 新元素对应的桶已满，需要分裂才可接受新元素，且Global Depth = Local Depth。这种情况不仅需分裂桶，也需扩容哈希索引数组。
  * 新元素对应的桶已满，需要分裂才可接受新元素，且Global Depth < Local Depth。这种情况仅需分裂桶。

由于哈希索引和哈希桶分裂后建立指针较为复杂，这里重点讲述。对于第二种情况，扩充的哈希索引大小与原先一样，直接附加于原先最后一个索引后面，设原先
哈希索引的大小为size，每个新哈希索引的指针指向的哈希桶应与下标为(索引下标 - size)的索引指向的哈希桶一致。在完成上述步骤后，哈希索引数组的
扩容便结束了。

之后便是第二，第三种情况都会经历的分裂桶步骤。对于新插入的元素，首先找到其对应的哈希索引下标，设为index，对index指向的哈希桶Local Depth加一，
取出桶中所有元素，放入一个链表list，再将桶清空，为之后重分配元素做好准备。
之后index索引指向新建的哈希桶(Local Depth设为上述增加后的值)。这时的关键是找到所有与index索引共享新bucket的索引(称之为兄弟索引)。

<figure>
    <a href="/assets/images/extendiblehashing.jpg"><img src="/assets/images/extendiblehashing.jpg"></a>
</figure>

如上图所示，图中每个哈希桶的Local Depth决定了参考索引的位数，Local Depth为3时，所有的索引位都起作用；Local Depth为1时，只有最后一位索引起作用。因此，寻找兄弟索引，最重要的是先确定起作用的索引位，将index索引位的数字提取出来，作为suffix，与其他索引比对，相同的便为兄弟索引。
上图中，所有Local Depth为3的索引均无兄弟索引，而Local Depth为1的索引有001，011，101，111四个，它们的suffix为1。

找到所有兄弟索引后，便可重新分配list里的元素了。分配完毕后再插入新元素。需要注意的是，一次桶分裂后新元素不一定能成功插入(原先的桶元素数量没变)，这种情况下需再次分裂桶，直至可成功插入为止。因此上述步骤应用while循环括住。

伪代码如下：

```c++
template <typename K, typename V>
void ExtendibleHashTable<K, V>::Insert(const K &key, const V &value) {
  std::scoped_lock<std::mutex> lock(latch_);
  size_t index = IndexOf(key);
  auto bucket = dir_[index];

  // use while loop to prevent insufficient expansion
  while (!bucket->Insert(key, value)) {
    // double the dir size
    if (bucket->GetDepth() == global_depth_) {
      // double the size of directory
      ......
    }

    bucket->IncrementDepth();

    // store items from the designated bucket
    auto items = bucket->GetItems();
    int local_depth = bucket->GetDepth();
    bucket->Clear();

    // create a new bucket which is pointed by the designated bucket
    dir_[index] = std::make_shared<Bucket>(bucket_size_, local_depth);
    num_buckets_++;

    // local_suffix means the suffix of the designated bucket, which is used to filter the sibling buckets
    size_t local_suffix = IndexOf(key) & ((1 << local_depth) - 1);

    // use local_suffix to find sibling buckets
    ......

    for (const auto &item : items) {
      // redistribute KV pairs
      dir_[IndexOf(item.first)]->Insert(item.first, item.second);
    }

    index = IndexOf(key);
    bucket = dir_[index];
  }
}
```

一些感想：

  * 可扩展哈希表的insert函数是本实验的难点，本人第一次实现时不加深入思索便写完了函数，最后发现没有考虑分裂桶次数大于1的场景，另外兄弟索引的寻找思路也不正确，只好重写。。。因此在写之前最好先动脑子把细节想清楚。
  * 第一次写完后跑本地测试用例发现会莫名奇妙地内存泄漏，找了半天未果，最后新桶改用智能指针完事。

## Task #2 - LRU-K Replacement Policy

大概是最简单的一个部分，原理并不复杂：将最新访问时间与k个时间戳之前的访问时间作差，选出最大的差，驱逐之。注意若有时间戳总量小于k的，应当将其差设为正无穷；而对于多个正无穷的情形，选择时间戳最早的驱逐。

Evict的伪代码如下：(其中table_evictable_为记录是否可驱逐的bool数组，table_为存贮元素的数组)

```c++
auto LRUKReplacer::Evict(frame_id_t *frame_id) -> bool {
  std::scoped_lock<std::mutex> lock(latch_);
  if (curr_size_ == 0) {
    return false;
  }

  size_t maxtime = 0;
  size_t earliesttime = std::numeric_limits<size_t>::max();
  std::unordered_map<frame_id_t, std::unique_ptr<std::list<size_t>>>::iterator i;

  for (auto it = table_.begin(); it != table_.end(); it++) {
    if (!table_evictable_[it->first]) {
      continue;
    }  // must be evictable
    // find the evictable iterator : i
    ......
  }

  *frame_id = i->first;
  table_.erase(i);
  table_evictable_.erase(*frame_id);
  curr_size_--;
  return true;
}
```

在写这一部分时和第一部分一样，也碰到了内存泄漏问题，同样改用智能指针完事。

## Task #3 - Buffer Pool Manager Instance

这一部分的难点在于弄清楚各变量之间的联系：

  * pool_size_ : buffer pool的总大小，不会变动
  * next_page_id_ : 下一个被分配的页id，只被AllocatePage()使用
  * bucket_size_ : page_table_的桶大小
  * pages_ : 一个存储page的数组，大小为pool_size_，每一个frame_id对应的page为pages_[frame_id] 
  * disk_manager_ : 负责向磁盘写入/写出页
  * log_manager_ : 无关紧要
  * replacer_ : 一个LRUK替换器，负责选出被驱逐的页
  * free_list_ : 存储空闲frame_id的链表，一个frame对应一个page
  * latch_ : 锁

一开始完成这部分时我没有意识到frame_id为page在pages_中的索引这件事，因此在写第一个函数时没办法通过frame_id找到对应的page，后来参考了其他人的博客，才发现是自己想错了。。。

这部分实现逻辑并不复杂，但比较繁琐，简单讲一下各函数实现思路:
  
  * NewPgImp : 先取得frame_id，在AllocatePage，将frame_id与page_id对应。注意取得frame是先从free_list_找现成的，若无，再调用replacer_，找到dirty的frame时先向磁盘写入
  * FetchPgImp : 与上面类似，区别在于要先从buffer pool中寻找指定的page，若无，则新增一个page
  * UnpinPgImp : 找到page后将pincount减一，注意设置SetEvictable
  * FlushPgImp : 向磁盘写入指定的page，写完后设置dirty位
  * FlushAllPgsImp : 向磁盘写入所有的page，写完后设置dirty位
  * DeletePgImp : 删除指定的page，注意DeallocatePage