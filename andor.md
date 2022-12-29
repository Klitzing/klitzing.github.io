---
title: "锐评《安多》"  
excerpt: "什么叫最好的星战衍生剧啊（战术后仰"
classes: wide
share: false
toc: true
toc_label: "目录"
---

本文对**[MIMIC数据集](https://physionet.org/content/mimiciii/1.4/)**进行了分析。

MIMIC数据库中包含了多种类型ICU（外科监护室、内科监护室、创伤外科监护室、新生儿监护室、心脏病监护室、心外恢复监护室）。MIMIC－Ⅲ数据集主要包括波形数据集（病人的生命体征趋势图）和临床数据集，按照记录内容的不同，共包含以下21个数据表：住院表、出院表、当前使用医疗服务记录表（CPT）、日期型事件表、医务人员表、监测情况表、ICD病情确诊表、诊断相关组编码表（DRG）、ICU记录表、注射记录表（CV）、注射记录表（MV）、排泄记录表、化验记录表、微生物检测记录表、文本报告记录表、病人登记表、处方信息表、过程事件表（MV）、ICD手术记录表、服务表、病房转移表。同时，数据集中还包含了５个辅助表用来辅助查找：目前使用医疗服务术语表、ICD病情确诊词典表、ICD医疗过程词典表、ICU化验词典表、门诊化验词典表。在对26个数据表的内容充分了解后，按照各个表的内容相关程度可分为四类，分别是病人基本信息及转移信息表、病人医院门诊的治疗相关信息表、病人在ICU里的治疗相关信息表和辅助信息表。

这个数据集主要关于病人的存活情况研究，因此我们关注根据患者入院情况，对其是否能够存活进行研究。

本文的研究由3部分构成，分别利用神经网络模型，集成模型和支持向量机模型构成。本文分别利用这3种方式进行研究，并将结果进行对比，来得到一个最好的患者存活预测模型。

因为本文数据缺失严重，因此，对于神经网络模型，集成模型，本文利用完整的数据和一些有缺失项的数据进行训练，如1，2，3列完全，第四列的6，7，8行完整，则我们选择数据集的1，2，3，4行的6，7行进行训练，再用8行进行检测拟合效果，最后把4列的缺失值也填补了，但是如果拟合效果太差，就会放弃这一列。同时，因为部分数据和我们的预测目标具有一致性，比如出院时间，因此本文的模型中删除了这些项。

## 第一节：神经网络模型

在本节中，利用了基础的神经网络BP神经网络进行建模，数据的预处理：神经网络的数据预处理一般是归一化，归一化是把输入数据映射到[-1,1]。

### 模型构建：

1.BP神经网络概念：

首先从名称中可以看出，Bp神经网络可以分为两个部分，bp和神经网络。bp是 Back Propagation 的简写 ，意思是反向传播。

BP网络能学习和存贮大量的输入-输出模式映射关系，而无需事前揭示描述这种映射关系的数学方程。**它的学习规则是使用最速下降法，通过反向传播来不断调整网络的权值和阈值，使网络的误差平方和最小。**

其主要的特点是：**信号是正向传播的，而误差是反向传播的。**

举一个例子，某厂商生产一种产品，投放到市场之后得到了消费者的反馈，根据消费者的反馈，厂商对产品进一步升级，优化，一直循环往复，直到实现最终目的——生产出让消费者更满意的产品。产品投放就是“信号前向传播”，消费者的反馈就是“误差反向传播”。**这就是BP神经网络的核心。**

2.算法流程图:

<figure>
    <a href="/assets/images/1114_1.jpg"><img src="/assets/images/1114_1.jpg"></a>
</figure>

3.神经元模型:

<figure>
    <a href="/assets/images/1114_2.jpg"><img src="/assets/images/1114_2.jpg"></a>
</figure>

每个神经元都接受来自其它神经元的输入信号，每个信号都通过一个带有权重的连接传递，神经元把这些信号加起来得到一个总输入值，然后将总输入值与神经元的阈值进行对比（模拟阈值电位），然后通过一个“激活函数”处理得到最终的输出（模拟细胞的激活），这个输出又会作为之后神经元的输入一层一层传递下去。

4.激活函数

<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width">
  <script type="text/javascript" async
  src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.5/MathJax.js?config=TeX-MML-AM_CHTML" async>
</script>
</head>
<body>
<p>
引入激活函数的目的是在模型中引入非线性。如果没有激活函数（其实相当于激励函数是\(f(x) = x\)），那么无论你的神经网络有多少层，最终都是一个线性映射，那么网络的逼近能力就相当有限，单纯的线性映射无法解决线性不可分问题。正因为上面的原因，我们决定引入非线性函数作为激励函数，这样深层神经网络表达能力就更加强大。
</p>
</body>
</html>

BP神经网络算法常用的激活函数：
 * Sigmoid（logistic），也称为S型生长曲线，函数在用于分类器时，效果更好。
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width">
  <script type="text/javascript" async
  src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.5/MathJax.js?config=TeX-MML-AM_CHTML" async>
</script>
</head>
<body>
<p>
    $$ f(x)=\frac{1}{1+e^x} $$
</p>
</body>
</html>

 * Tanh函数（双曲正切函数），解决了logistic中心不为0的缺点，但依旧有梯度易消失的缺点。
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width">
  <script type="text/javascript" async
  src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.5/MathJax.js?config=TeX-MML-AM_CHTML" async>
</script>
</head>
<body>
<p>
    $$ f(x)=\frac{e^x-e^-x}{e^x+e^-x}  $$
</p>
</body>
</html>

* relu函数是一个通用的激活函数，针对Sigmoid函数和tanh的缺点进行改进的，目前在大多数情况下使用。
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width">
  <script type="text/javascript" async
  src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.5/MathJax.js?config=TeX-MML-AM_CHTML" async>
</script>
</head>
<body>
<p>
    $$ f(x)=max(0,x)  $$
</p>
</body>
</html>

5.神经网络基础架构

BP网络由输入层、隐藏层、输出层组成。

* 输入层：信息的输入端，是读入你输入的数据的
* 隐藏层：信息的处理端，可以设置这个隐藏层的层数（在这里一层隐藏层，q个神经元）
* 输出层：信息的输出端，也就是我们要的结果

v，w分别的输入层到隐藏层，隐藏层到输出层的是权重

对于下图的只含一个隐层的神经网络模型：BP神经网络的过程主要分为两个阶段，第一阶段是信号的正向传播，从输入层经过隐含层，最后到达输出层；第二阶段是误差的反向传播，从输出层到隐含层，最后到输入层，依次调节隐含层到输出层的权重和偏置，输入层到隐含层的权重和偏置。

<figure>
    <a href="/assets/images/1114_3.jpg"><img src="/assets/images/1114_3.jpg"></a>
</figure>

6.神经网络的训练

模型优化器（AdamW算法）

训练数据集具有类别间差异大的特点，这使得本文在模型优化过程中可能会面对目标函数不够稳定，部分模型参数难以收敛的问题。

一阶优化算法已被实际用于训练各种机器学习模型，它具有较低的计算复杂度和较高的拟合优度。国内外相关研究中提到的随机梯度下降 (SGD) 就是一种一阶优化算法，它每次提取小批量的样本计算损失函数的梯度来迭代模型参数，但是无法直接优化不稳定的损失函数。

因为本文的目标函数是不稳定的，为此本文借鉴了一些自适应方法的思想，此类方法可以使用不同的学习率迭代不同的参数。AdaGrad算法是最早被提出的自适应方法，RMSProp算法和ADADELTA算法则是随后被提出的针对AdaGrad的改良方法，这两个算法将AdaGrad算法的历次梯度和改变为了历次梯度平均值以解决AdaGrad算法若干次迭代后校正学习率过小不能有效迭代的问题，Adam方法则是AdaGrad算法和RMSProp算法的结合。凭借其出色的性能，Adam方法一度在训练深度神经网络中有着广泛而实际的应用。然而，如果梯度和动量估计之间不存在较强的相关性，则Adam优化器无法收敛到最优解，且在2018年ICLR的最佳论文: **[On the Convergence of Adam and Beyond](https://openreview.net/pdf?id=ryQu7f-RZ)**中研究者发现Adam的收敛性证明存在问题。因此本文采用了AdamW算法进行训练，AdamW算法与Adam算法类似，但是AdamW方法将Adam的weight decay改为了L_2正则化,这种做法有效地避免了Adam算法无法收敛到最优解的问题。

AdamW算法实现简单，对内存需求不高，也适用于不稳定目标函数和存在很大噪声的样本，因此很适合被应用于本文的大规模参数场景。

AdamW算法将自适应学习率与动量项有机结合，整合了梯度的一阶矩估计（First Moment Estimation，梯度均值）和二阶矩估计（Second Moment Estimation，梯度方差），计算出更新步长。

AdamW更新算法具体步骤如下：

<figure>
    <a href="/assets/images/1114_4.jpg"><img src="/assets/images/1114_4.jpg"></a>
</figure>
