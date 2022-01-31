# genshin-artifact-up-model

A well-visualized Genshin Impact artifact upgrade model with classifier (decide whether to upgrade or not)

原神圣遗物强化模型，使用二元分类器决策是否升级，拥有多种可视化方案

## 前言(preface)

### 依赖(Requirements)

```python
numpy
matplotlib
```

### 名词缩写(abbreviation)

```list
    "ATK": "小攻击",
    "ATK_P": "攻击%",
    "DEF": "小防御",
    "DEF_P": "防御%",
    "HP": "小生命",
    "HP_P": "生命%",
    "ER": "元素充能",
    "EM": "元素精通",
    "CR": "暴击率",
    "CD": "暴击伤害",
    "main stat": "主词条",
    "sub stat": "副词条",
    "sets": "套装",
    "flower": "花",
    "plume": "羽",
    "sands": "沙",
    "goblet": "杯",
    "circlet": "冠",
    "HEAL_BONUS": "治疗加成",
    "PRYO_DMG": "火伤",
    "ELECTRO_DMG": "雷伤",
    "CRYO_DMG": "冰伤",
    "HYDRO_DMG": "水伤",
    "ANEMO_DMG": "风伤",
    "GEO_DMG": "岩伤",
    "PHYSICAL_DMG": "物理伤",
    "ELEM_DMG": "元素伤害加成"
```

## 原理

分为数学原理与模型机制来介绍

### 数学原理

#### 加权随机取样(Weighted Random Sampling)

圣遗物词条抽取的过程是加权随机抽样

* 加权随机抽样问题指从n个带权元素集合中按权重抽取m个元素

伪代码实现如下

```pseudo-code
1:for k=1 to m do
2:    pick vi from weight table
      (p_i(k)=w_i/sum(w_j) where j is the element in the weight table)
3:    pop vi from weight table
4: end for
```

具体到原神圣遗物抽取中

```pseudo-code
1:从位置中抽取POS
2:从对应位置POS的主属性中抽取MAIN
3:    获取对应权值表T
4:抽取初始词条N(3 or 4)
5:for k=1 to N do
6:    从权值表T中选择词条S
7:    将权值表T中的S项去除
8:end for
```

例如以下展示一次加权随机取样在圣遗物抽取中的实现

```
1:从位置中抽取到“花”
2:从“花”对应的主属性中抽取到“小生命”
3：抽取到初始4词条
4:when k=1 do
5:    从权值表中抽取
      （比如抽到“暴击率”，权值75，原总权值=1100-150（小生命主词条）=950，对应概率=75/950）
6:    将原来权值表中的“暴击率”一项删去
7:when k=2 do
8:    从权值表中抽取
      （比如抽到“元素精通”，权值100，原总权值=1100-150-75（暴击率的）=875，对应概率=100/875）
9:...
```

![simplified model](./doc/graph/WRS.jpg)

> (figure1) Weighted Random Sampling in GI artifact generation

权值表(figure2)我们已经基本知道[1][2]，因此我们可以用计算机模拟这一过程。

算出

1. 在3初始词条圣遗物中，各词条在所有词条中出现的概率(figure3)
2. 在4初始词条圣遗物中，各词条在所有词条中出现的概率(figure4)
3. 在初始圣遗物中，各词条在所有圣遗物中出现的概率(figure5)
4. 在完成升级的圣遗物中，各词条在所有圣遗物中出现的概率(figure6)

值得注意的是(4)中的概率其实就是(2)中概率的4倍，即p4=p2 * 4，从定义上这也是容易理解的；而(3)中概率就是(2)和(1)的乘以对应词条的加权平均，即p3=p1 * 3 * 0.8+p2 * 4 * 0.2

数据验证

结果与[2]中的/Distribution和[3]的结果一致

![weight table](./doc/graph/weight.jpg)

> (figure2) weight table 权值表

![figure3](./doc/graph/init3.jpg)

> (figure3) 在3初始词条圣遗物中，各词条在所有词条中出现的概率

![figure4](./doc/graph/init4.jpg)

> (figure4) 在4初始词条圣遗物中，各词条在所有词条中出现的概率

![figure5](./doc/graph/init.jpg)

> (figure5) 在初始圣遗物中，各词条在所有圣遗物中出现的概率

![figure6](./doc/graph/finish.jpg)

> (figure6) 在完成升级的圣遗物中，各词条在所有圣遗物中出现的概率

参考资料：

[1][github.com/Dimbreath/GenshinData](https://github.com/Dimbreath/GenshinData/blob/master/ExcelBinOutput/ReliquaryMainPropExcelConfigData.json)

[2][Fandom Wiki/Genshin Wiki/Artifacts](https://genshin-impact.fandom.com/wiki/Artifacts/)

[3]小明明明中观察. [【数据讨论】 【提瓦特大学】(附代码)圣遗物副词条与其中的多重概率问题](https://bbs.nga.cn/read.php?tid=26589982) .[OL].2021.5.14

#### 多项分布(Multinomial Distribution)

从强化次数或有效词条到具体数值的过程是多项分布

* 多项分布是二项分布的扩展，其中随机试验的结果不是两种状态，而是K种互斥的离散状态，每种状态出现的概率为pi，p1 + p1 + … + pK = 1，在这个前提下共进行了N次试验，用x1-xK表示每种状态出现次数，x1 + x2 + …+ xK = N，称X=(x1, x2, …, xK)服从多项分布，记作X-PN(N：p1, p2,…,pn)

我们知道圣遗物具体数值分为4个等级，它们之间的比例为7:8:9:10，不妨将强化值设为(7,8,9,10)。圣遗物词条数为3到9，因此词条数为n的圣遗物数值分布在[7n, 10n]

例如：暴击率分布挡位是（2.72, 3.11, 3.50, 3.89）其比例就是7:8:9:10，那么我们把+3.89定义为+10，把+3.50定义为+9，依次类推

由此通过穷举法，算得不同词条数能取得对应值的概率分布

纵轴是有效词条（强化了多少次），横轴是强化出来对应的值（上文例子中的表示），色块上的值是在某有效词条下，强化出对应的值的概率

![figure7](./doc/graph/s3456.jpg)

> (figure7) 词条数=3，4，5，6时的实际值分布

![figure8](./doc/graph/s789.jpg)

> (figure8) 词条数=7，8，9时的实际值分布

#### 分类器(Classifier)

在一般讨论中，我们常用**线性函数**表示圣遗物的价值，并对超过阈值的圣遗物进行强化

因此在强化中使用**二元分类器**判断是否进行强化是合理且有效的

令分类器为w，圣遗物为a，副词条有10种，添加一个常数项，使其变为11维矢量

$w,a\in R^{11}$

$f(a)=w^Ta$

则强化条件为$f(a)=w^Ta>0$

本模型中使用逻辑回归(Logistic Regression)得到分类器

### 模型机制

#### 简化模型

一般的讨论有些是基于**简化模型**的讨论，该模型有如下特点：

1. 评价标准只适用于初始强化
2. 中途强化过程无评价标准
3. 不对具体位置进行讨论

这将导致以下结果：

1. 强化次数和所得圣遗物数量远大于实际，体现为过快到达退出条件
2. 具体套装和位置不做区分，使退出条件模糊不清

![simplified model](./doc/graph/simp_model.jpg)

> (figure9) 简化模型

#### 本项目模型

项目使用的模型在简化模型的基础上做出如下优化：

1. 对每一级强化，每一个位置，使用不同的分类器进行判断
2. 明确圣遗物仓库中最优组合的定义
3. 使用线性函数规定退出条件
