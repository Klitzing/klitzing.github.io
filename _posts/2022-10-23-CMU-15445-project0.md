---
title: "CMU 15-445 Project0 总结"  
excerpt: "一些C++基本功"
classes: wide
header:
  image: /assets/images/15445pro0.jpg
share: false
---

**[CMU 15-445](https://15445.courses.cs.cmu.edu/fall2022/)**是CMU关于数据库设计和实现的一门基础课程，对全校的本科生和研究生开放。2022年这门课由数据库领域的大牛 **[Andy Pavlo](http://www.cs.cmu.edu/~pavlo/)**讲授，课程内容涵盖存储引擎，索引模型，查询优化和并发运算等多方面知识，非常适合数据库初学者将其作为数据库的第一门课程学习。

本课程（2022年版）一共有5个Project，其中本次要完成的Project0为对C++基础知识的检测。
对C++不甚了解的初学者可能会感到有难度。所以在介绍实验细节之前先将会用到的C++知识进行汇总：

  * 右值引用
  * 移动语义(move和forward函数)
  * 智能指针
  * 类型转换

## C++ 基础知识

### 右值引用
先把参考资料列举出来：**[一文读懂C++右值引用和std::move](https://zhuanlan.zhihu.com/p/335994370)**， 另外**C++ primer**的13.6节也对右值引用有基本的解释。

中心点：

  * 左值可以取地址，位于等号左边；右值没法取地址，位于等号右边
  * 左值引用能指向左值，不能指向右值（所以左值应用无法取常量）
  * 右值应用可以指向右值，不能指向左值
  * std::move()函数可将左值转换为右值

右值引用的好处是对于函数传参，以右值引用为参数可以避免深拷贝，提高效率(在本project中InsertChildNode函数采用右值引用将std::unique_ptr作为形参)。

### 移动语义
参考资料：**[C++11朝码夕解: move和forward](https://zhuanlan.zhihu.com/p/55856487)**, **[Usage of std::forward vs std::move](https://stackoverflow.com/questions/28828159/usage-of-stdforward-vs-stdmove)**。

中心点：

  * std::move()可将左值转换为右值
  * std::forward()对左值和右值同样适用，范围更广

### 智能指针
参考资料：**[What is a smart pointer and when should I use one?](https://stackoverflow.com/questions/106508/what-is-a-smart-pointer-and-when-should-i-use-one)**, **C++ primer**12.1.4节。

中心点：

  * 智能指针是包装原始C++指针的类，用于管理所指向对象的生命周期
  * std::unique_ptr无法被复制，不过其引用可以传递给其他参数
  * std::shared_ptr可以被复制，当指向同一对象的所有指针生命期均结束后，对象被删除
  * std::weak_ptr指向std::shared_ptr，其本身不计入引用数，当所有shared_ptr生命期结束后，weak_ptr指向对象自动消失

### 类型转换
参考资料：**[dynamic_cast and static_cast in C++](https://stackoverflow.com/questions/2253168/dynamic-cast-and-static-cast-in-c)**

中心点：

  * static_cast可以由基类转换到派生类，或由派生类转换到基类，如果转换不合法，将得到nullptr

## Project实现

本实验要求实现一个**[字典树结构](https://en.wikipedia.org/wiki/Trie)**，包括基本的数据结构实现和并发控制。
对于字典树，参考**[leetcode 208](https://leetcode.com/problems/implement-trie-prefix-tree/)**的实现方法。

本实验提供了三个类：TrieNode Class，TrieNodeWithValue Class，Trie Class

  * TrieNode Class代表单个非叶节点
  * TrieNodeWithValue Class代表叶节点
  * Trie Class代表字典树总体，其中根节点是以'\0'为值的TrieNode

由于课程要求，具体的实现代码不能公开，因此在这篇文章中仅介绍思路。

对于TrieNode Class，实现上没有难点，按照注释去做就行。需要注意的点：

1. 遇到右值引用需要调用move函数
```c++
children_ = std::move(other_trie_node.children_);
```

对于TrieNodeWithValue：

2. 创建构造函数时，利用基类的构造函数(注意TrieNode类构造函数的参数为右值引用，需要利用move或forward函数)
```c++
TrieNodeWithValue(TrieNode &&trieNode, T value) : TrieNode(std::move(trieNode)) {
  this->value_ = value;
  this->is_end_ = true;
}
```

对于Trie：

3. 完成Insert函数时，伪代码如下：
```c++
size_t i = 0;
while (i < key.size() - 1) {
  若有key[i]，跳过；
  若无，植入key[i]为值的TrieNode
}
// 最后一个字母
if (!(*ptr)->HasChild(key[i])) {// 1. 含有此字母的节点不存在
  ......
} else if ((*((*ptr)->GetChildNode(key[i])))->IsEndNode() == false) {//2. 含有此字母的节点不存在为TrieNode
  ......
} else {//3. 含有此字母的节点为TrieNodeWithValue,
  ......
}
```

4. 对于Remove
```c++
//创建stack,记录找到对应节点的路径
std::stack<std::unique_ptr<TrieNode>*> mystack;
mystack.push(ptr);
while (i < key.size()) {
  找到尾节点
}
std::unique_ptr<TrieNode>* last, *last_2;
last = mystack.top();
mystack.pop();
while (mystack.size() != 0) {
  若无子节点，删除之，继续向上删除
}
```

5. 对于GetValue
```c++
if (key.size() == 0) {
  *success = false;
  return T();      
}
auto ptr = &root_;
size_t i = 0;
while (i < key.size()) {
  找到以key为字符串的节点链
}
//类型转换，需要注意的是：(*ptr)为unique_ptr,(*(*ptr))为TrieNodeWithValue对象，(&(*(*ptr)))为指向TrieNodeWithValue对象的普通指针
if (dynamic_cast<TrieNodeWithValue<T>*>(&(*(*ptr))) == nullptr) {
  *success = false;
  return T();   
} else {
  *success = true;
  return dynamic_cast<TrieNodeWithValue<T> *>(&(*(*ptr)))->GetValue();
}
```

对于并发控制：

在Insert，Remove，GetValue的开头加锁，return之前解锁即可























