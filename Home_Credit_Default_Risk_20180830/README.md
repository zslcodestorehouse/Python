    
在做腾讯广告赛的时候，刚开始学数据挖掘和py，程序写的比较慢，过了初赛时间没来得及提交成绩，没进复赛，就到处找比赛做。刚好看到这个比赛开始了一个月多点的时间，又是自己准备深入学习的推荐和金融两个方向中的金融风控比赛，果断开始加入比赛刷分，一开始单纯就是想学习，刷到后面各种学习大佬们分享的kernel和心得，就开始朝着前10%努力了，最后pb成绩出来前进了200多名到了300名左右，也是意外之喜，再次证明，local cv很重要！！感谢kaggle社区大佬的分享和Neptune开源项目，学到了很多特征技巧。下面将文件夹中的所有文件，都记录下编写的原因和用处。同时记录下比赛中的经验，方便以后再次学习查看。

### 1.数据分析

    
   data_analysis文件夹中all_data_draw_tool.py是对所有数据进行可视化的函数，文件夹里面的图片都是用这个文件生成的，visual里面有部分图片以及所有图片的打包。这部分代码主要是为了初步的对数据进行查看，配合HomeCredit_columns_descriptio(中译).csv中对变量的描述，了解比赛举办方提供了一些什么类型和含义的数据，正负样本的数量，以及数据的分布情况，缺失值的分布情况，离群点的查看，变量类型，取值范围等等。

### 2．数据清洗和特征工程

    
   1_pos_hand_craft.py,1_pre_app_hand_craft.py,2_ins_hand_craft.py,3_feature_engineering.py, 3_feature_engineering.py,这几个文件，包含了数据的清洗和特征工程。数据的清洗主要是异常值的处理，变换到合理范围或者丢弃，或者替换为nan，然后对类别特征one-hot。特征工程包括对相关原始特征的加减乘除，所有表的各种聚合统计特征，趋势特征，以及一些模型预测值做为特征（这部分特征是根据赛后一些金牌队伍分享的心得做的实现，比赛最后提交的成绩并未用到，根据理解做了代码实现并加入模型，但是效果没有在他们模型中那么好，相对于最终的比赛成绩只有些许提升）。特征工程是整个比赛最重要的部分，基本上特征工程就决定了模型的分数。金融风控不知道有哪些特征与违约强相关，所以一开始只是加了一些加减乘除和统计类特征，全部弄完合并到train表然后训练了第一个结果，得到.784，后面一些特征基本上就是看kernel的分享和讨论区的讨论里面各种提取别人的想法然后往模型里面加了，不得不说，kaggle的讨论区其实在后期提分比kernel重要的多，毕竟kernel大家都能copy后直接用，讨论区里面很多得从别人的言谈里面推敲，然后再去实现，这样对于名次提升还是明显得多。还有很多专业人士会分享一些专业领域的业务理解，这对于做特征还是很有帮助的。另外，有些特征随时间变化，统计越靠近现在的特征对模型的效果越强。

### 3. 特征筛选
 

   4_feature_select.py，4_feature_select_ridge_2这两个是对特征做筛选。一开始单纯的用lightgbm的特征重要度筛选特征，效果并不理想，筛选完特征cv就有下降，而且特征维度还是很巨大。这次比赛提供的数据，其实总大小才1g多，但是做完特征工程到程序里面跑我的16gb的电脑基本上内存吃满，因为特征很容易就2,3000+。4_feature_select.py是比赛期间使用的筛选特征的方式之一，筛选完特征大概2000+，导致训练很慢，还有另一种筛选方式，后面会说，但是比赛后期时间不够也就没有再往这方面做优化。4_feature_select_ridge_2这是赛后才加入的一种特征筛选方法，简单有效。主要思想是对特征使用lightgbm初步训练一遍，得到特征重要度排名。然后，从最重要的特征开始，每次加入一个特征使用ridge进行5-kfold训练和预测，full auc增加则保留此特征，循环所有特征即可。通过这种方式筛选特征后，2000+特征减少到600+，同时cv提升差不多0.001，pb也是。

### 4.模型训练，blending，stacking


   单模型选择lightgbm，cv大概在0.797，lb在0.802。因为特征工程感觉做不动了，就准备进行多模型融合，看能不能提高点成绩。一开始是用stacking做的，5_train_single_and_stacking中是这种方式的实现，不过就是用的相同的特征做了5个不同模型的oof，然后堆叠做为二级模型的输入训练，二级模型尝试过lr和xgb，xgb有不到0.001的cv提升，LB上面没有提升，所有也就没有继续使用了。后面看到很多人的分享，简单的stacking在这个比赛里面确实没什么提升，还是要选择不同的特征子集，不同的模型才能有多样化的效果。5_train_blending.py ，6_Bayesian_optimization_blending，这是使用贝叶斯优化的方式进行融合，这次考虑到stacking用相同的特征训练集没有效果，在kernel找了一种null importance的特征筛选方式，筛选了不同的子集然后训练模型融合，cv和lb也都提升了0.001，但是后期加了很多新特征以后提升基本上看不到了。总结来说，模型的多样化还是不够，其实本应该随机选择特征子集，然后随机选择kfold种子数和不同模型随机参数训练一堆模型然后再融合的，但是自己电脑实在跑不动，公司电脑也没那么多时间给我用来跑程序，也就只能搭了一个这个blending的架子用来融合试试了，看大佬们都是上百个模型融合，只能围观下了。

### 5.文件运行顺序（文件中有路径需要根据自己的数据放置位置进行修改）


   1_pos_hand_craft.py->1_pre_app_hand_craft.py->2_ins_hand_craft.py->3_feature_engineering.py->4_feature_select.py->4_feature_select_ridge_2
以上运行完毕得到训练特征集，后续有两种方式：

5_train_blending.py ->6_Bayesian_optimization_blending

5_train_single_and_stacking
最终提交cv0.7985, LB0.80361, 之前的部分训练结果在训练结果记录.txt中。

### 记录下冠军队伍分享中比较好的idea：


 **1th**   DAE+NN，用神经网络训练的结果虽然没有lightgbm训练的好，但是作为stacking的一级模型，用来增加模型的多样化非常有效，能够学到许多变量间的关系。Stacking可以超过2层，他们用了3层，第二层NN, ExtraTree ，Hill Climber，第三层次用等权值混合。

 **2th**   非常清晰的流程图。
![加载图片失败](https://github.com/AiIsBetter/Kaggle/blob/master/Home_Credit_Default_Risk_20180830/IMG/1.png)

 **4th**    对100+oof在做stacking第二层之前，用lightgbm做了一次筛选，对cv有提升。

 **5th**    1.用dp训练以前贷款的数据，找出以前各个表中的交互，这个作者还在写详细细节，后续再补充。2.用现在贷款的target，用SK_ID_CURR 合并到以前的各个贷款记录和在信贷局的记录表中（pre_,bur_）,训练模型，用以前的数据预测target，得到的预测值再返回app表作为训练特征。这个预测值用来表示以前的信贷行为对于现在贷款行为的影响，相比于传统的用权值表达行为的影响大小，这个通过模型学习更加自动化，不用人工设定。

 **8th**  stacking第一层的oof，可以做为特征合并到原始特征集中用来做为新特征训练第二层，下图是完整流程，这个融合值得借鉴。
![加载图片失败](https://github.com/AiIsBetter/Kaggle/blob/master/Home_Credit_Default_Risk_20180830/IMG/2.png)

### 总结：

kaggle是一个非常好的大数据竞赛网站，里面有众多大佬非常热衷对新手分享入门级的完美kernel并且附带详细的解释。同时他们做数据挖掘的流程和思想有太多值得思考和学习的地方。刷了这2个月分，感觉学到了太多之前几个数据比赛没学到的东西，从数据分析、清洗、挖掘建模、模型融合，都完整实战了一次。欠缺的地方，一是特征工程没做好，没有挖掘到强特征，对风控这块业务特征不了解，基本上靠瞎想加猜、试，二是最终提交的是单模型结果，没有做多模型融合，组队获得不同特征训练的模型应该是一个获得模型差异性比较好的方法。
赛后对于金牌队伍的一些好的想法进行了实现并加入模型，确实相对于最终成绩有提升，还有些类似融合和nn的方法暂时还没有做，以后有时间有条件还是要尝试实现一下看看具体的效果。

### 引用：
1.https://www.kaggle.com/ogrellier/feature-selection-with-null-importances/notebook

2.https://www.kaggle.com/aantonova/797-lgbm-and-bayesian-optimization

3.https://www.kaggle.com/c/home-credit-default-risk/discussion/57175

4.https://www.kaggle.com/c/home-credit-default-risk/discussion/64510

