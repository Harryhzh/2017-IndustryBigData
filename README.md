
#### 10.24更新： 发现很多同学都在研究这个问题，现已把代码更新为可以直接运行的代码。

如果本文对您有用，可以给我一个star。

数据集：链接: https://pan.baidu.com/s/1k8OIB6sBag43fvKtFb3vWQ  密码: tpov

----


写在开头：老师给了数据集，主要练习对时间序列的操作，本文内容适合机器学习初学者入门借鉴，有一套较为完整的从数据预处理至模型评价的流程。

>  **文章简单总结了一些风机叶片预测问题中需要解决的问题与部分解决方案（采用的是基础的随机森林模型）。**
>
> **解决的问题基于ypjbyc/01_叶片结冰预测/train/15文件夹中的39.39万条数据。**

#### 数据集的使用说明： 

在ypjbyc/01_叶片结冰预测/train/15/中分别由三个csv文件：

- 15_data.csv：风机叶片数据集，包含time和其他27个特征。其中time是每一条数据的时间，需要依据下面两个时间表，对每一条数据是故障或是正常或是无效数据进行判断。

- 15_failureInfo：故障风机叶片时间数据，可以据此确定15_data中每一条故障数据的标签。

- 15_normalInfo：正常工作风机叶片时间数据，可以据此确定15_data中每一条正常工作数据的标签。

- **15_data_label为已经对上述工作操作完毕后的csv表格。如果觉得上述步骤分析无误可以直接使用15_data_label。由于在对时间进行标签后，时间数据可能不再有意义，15_data_label2为确定label后删除time列的表格，可供使用。**

---

#### 叶片预测数据集中需要解决的问题：

- 样本中存在停机、人为删除数据、无效数据等，会造成某些时间段的数据缺失 ，如何对缺失数据与数据不均匀问题进行处理？此外，叶片正常(normalInfo)与结冰故障(failureInfo)数据不均衡，该如何处理？
- 选取哪些特征用于模型预测？
- 何种训练模型比较适合本问题？
- 针对上述问题，从论文中摘出的一些解决方案：

> 在建立预测分类模型的时候，需要考虑风机结冰数据的类 不平衡问题。一般来说，对数据进行重采样能够有效降低类不 平衡带来的建模误差。将结冰样本进行过采样，将非结冰样本 进行欠采样，或者两者同时进行，以达到结冰和非结冰样本在 模型训练时有基本相近的比例。 
>
> 如果我们能从序列数据中提取出易于分类的特征，就能够 准确地检测早期结冰。因此，在设计预测模型时，不仅要提供 瞬态特征，还要利用滑动窗提取给定长度下的统计特征。这些 统计特征能够很好地反映这一段序列数据的演化规律和状态， 因此能够更好的发现早期结冰。 
>
> 我们根据训练数据中的 group 维度随机 地删除部分非结冰数据以及结冰严重数据，经过剔除数据处理， 正负样本所占比依然存在很大的差异，这种数据不平衡会对模 型最终的叶片结冰预测有非常大的影响，一般处理方式有三种: 
>
> 欠采样、过采样、在模型 loss 函数中增加惩罚项以及模型集成。由于我们想尽可能的保留原始数据，因此选择第三种方法， 并且选取了更加适合此种情况的评价函数。 

---

### 问题解决（仅供参考）

#### Step1 :读取数据并标注每条数据的标签

- 故障时间区间覆盖的数据行标记为1。
- 正常时间区间覆盖的数据行标记为0。
- 无效数据不参与评价。
- 数据集描述：

```c
sum of data:393886
sum of failure data : 23846 , 6.05 % 
sum of normal data : 350255 , 88.92 % 
sum of invalid data : 19785 , 5.02 %
```

#### Step2:删除无效数据，选取此次要预测的数据（1/10），划分训练（2/3）与测试集（1/3）数据

- 删除19785条无效数据

- 剩下374101条有效数据

- 选取了1/10 39388个数据(乱序)

  - 25064训练集(66.66%)

  - 12346测试集(33.33%)

    训练集中：

    - failure:1627
    - normal:23437

#### Step3 特征分析

- 使用RFECV进行特征选择，以交叉验证分数高低决定选取的特征数量
- 对特征进行排名：

```python
Ranking of features names: ['wind_direction', 'pitch1_moto_tmp', 'wind_direction_mean',
       'pitch2_speed', 'pitch3_moto_tmp', 'pitch2_angle', 'acc_y',
       'pitch1_angle', 'pitch1_speed', 'yaw_speed', 'pitch2_ng5_DC',
       'pitch3_ng5_DC', 'group', 'pitch3_angle', 'pitch3_speed',
       'yaw_position', 'acc_x', 'pitch3_ng5_tmp', 'power', 'generator_speed',
       'int_tmp', 'pitch2_moto_tmp', 'pitch1_ng5_DC', 'pitch2_ng5_tmp',
       'environment_tmp', 'pitch1_ng5_tmp', 'wind_speed']
```

- 绘制交叉验证下的特征选择折线

![](https://img-blog.csdnimg.cn/20190515075559707.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3poYW9oYWlib18=,size_16,color_FFFFFF,t_70)

- 选取得分前三的特征，绘制特征对比图

![](https://img-blog.csdnimg.cn/20190515075610380.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3poYW9oYWlib18=,size_16,color_FFFFFF,t_70)

#### Step4 建立预测模型

- 这里选择的是随机森林分类模型
- 模型参数选择：
  - 选择树的最⼤大深度为146
  - 树的最⼤大节点数 2500
  - 节点拆分最小用力数量：2
  - 叶子结点最少样本数：2
  - 集成决策树个数：2500
  - 最少的叶子节点数：2500
  - Bagging中每个字模型每次放回抽样选取的样本个数
  - 当寻找最佳分割时要考虑的特征数量：sqrt
  -  评价模型标准：Out Of Bag

#### Step5模型评价

- **accuracy_score:0.9998005106926269（该结果可能与选取训练集的随机度不够高有关）**
- 混淆矩阵：

![](https://img-blog.csdnimg.cn/20190515075621220.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3poYW9oYWlib18=,size_16,color_FFFFFF,t_70)

- 评分

                     precision    recall  f1-score   support
    
                0       1.00      1.00      1.00     11549
                1       1.00      0.99      1.00       797
      avg / total       1.00      1.00      1.00     12346
