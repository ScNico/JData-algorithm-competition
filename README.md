### JD算法大赛

京东JData算法大赛-高潜用户购买意向预测 Rank24

#### 队伍成员

- 不觉晴光老
- Polonaise
- MKFMIKU
- fyxx
- o0Helloworld0o

#### 数据分析
该比赛以京东商城真实的脱敏后的用户数据，大概有10万条，包含了用户的年龄、性别、等级、注册时间等；商品数据，大概有2.5万条，包含了商品的属性、类别以及品牌；评价数据，约56万条，包含了评论数、差评率等；行为数据，大概有5000多万条，包含了用户对商品的行为，例如浏览、点击、加购、关注等。

<img src="http://p2l71rzd4.bkt.clouddn.com/blog-image/180317/2djHc1Ela2.jpg?imageslim" style="zoom:30%" />

总共两个半月的数据，我们要以这些数据为基础构建用户购买商品的预测模型。评价指标分为两个方面，第一个是预测未来一周哪些用户会购买目标类商品，第二个是预测未来一周发生购买的用户会购买目标类别的哪件商品。根据题目我们构建了两个模型，这两个模型都是用的xgboost，分别得到会产生购买的用户和购买的用户商品对。

#### 数据处理
首先是数据处理部分，通过统计我们发现购买目标类商品的用户的空档期很长，可能几个月都不会再次购买商品，所以就去掉了那些购买过的用户。然后去掉了一些冗余的数据以及爬虫数据，爬虫用户直接访问某个页面，点击数远小于浏览数。对于提取特征的区间我们选了预测期前60天用于提取特征，这样做会造成最后的训练数据集很大，训练起来有难度，所以我们在提取完特征后只保留了前一周有交互的数据作为训练数据，因为我们发现前60天有交互的在预测期会购买的占预测期总购买比大约是45%，而前一周大概是占35%，可以看出只选前一周并不会流失太多的用户；此外如果选前60天得到的训练集正负比大约是1:1100左右，而前一周的正负比就会降低到1:400；另外训练的数据量上来看，只选前一周有交互的数据量也更方便我们训练。最后得到的用户模型训练集大概6万条，用户商品模型训练集大概35万条。

#### 特征工程
其次是特征工程部分，因为一共有两个模型，而用户商品模型的特征包含了用户模型的特征，所以我主要讲用户商品模型的特征工程。我们将特征主要划分为3大部分，分别是用户特征、商品特征以及行为特征。下面从三个方面谈谈分别的特征工程，当然只会讲比较主要的特征。

##### 用户特征
用户特征方面主要分为两个方面，一是用户本身的属性特征，例如等级、年龄、注册日期距离预测期的时间（判断是不是老用户）；二是用户与商品交互产生的属于用户的特征，例如用户最后一次与目标类商品交互距离预测期的时间，用户在某个时间窗口的入度特征（商品访问图模型）、有效交互时间、对目标类的操作/总操作以及一些统计与时间衰减乘积和转化率等。具体讲一下时间窗口、时间衰减、商品访问图模型以及有效交互。

时间窗口用于对时序数据提取特征，简单的来说其实就是统计某个窗口内的数据，例如前60天的特征提取，我们可以分别提取前1、2、3、5、15、60天的特征。而时间衰减表达的是用户行为的有效性距离预测期越近，该行为越有效，例如一个用户前60天的第一天操作与第六十天操作当然是第六十天的操作对于用户产生购买更有效，那么我们怎么用数值体现这个有效性呢，我们通过各个时间窗口在预测期的购买用户数/该窗口总交互用户数便得到了时间衰减的特征，如下图所示。

<img src="http://p2l71rzd4.bkt.clouddn.com/blog-image/180318/5deHkdeFcc.png?imageslim" style="zoom:80%" />

然后是商品访问图模型，其表达的是用户对比该商品的次数，一个商品访问图表示的是一位用户对比商品的顺序。例如一位用户想购买一双鞋子，首先看到了鞋A，这时，很大可能该用户还会去浏览鞋B和鞋C，当用户第一次接触该商品时，入度就记为1，之后若用户在访问其他商品后再回来访问该商品，该商品的入度就增加1，若用户多次将一件商品与其他商品进行对比，可以理解为这件多次被对比的商品在用户的心中很大可能是目前为止用户接触过最好的商品，具体的可以结合图来理解。

<img src="http://p2l71rzd4.bkt.clouddn.com/blog-image/180317/Bfm6a8Cd2E.jpg?imageslim" style="zoom:100%" />

问题主要在我们怎么来构建这样一个图，由于数据量比较大，单独的把数据拿出来为每个用户构建这样一个图结构不太划算，所以我们从数据的角度去分析这个问题，我们首先将数据按照用户ID和时间排序，我们发现连续访问某一件商品并不能算入该商品的入度，而入度表达的是上次访问与本次访问的商品ID不同，所以我们选择将整个商品ID列向上移一行够成新的商品ID列，在末尾添加与商品ID无关的字符用于对其，这样我们就可以直接将新的商品ID与旧商品ID相等的行删去然后按照用户和商品groupby就可以统计出用户商品的入度。但这样我们发现如果上一个用户ID最后访问的商品ID与下一个用户第一件访问的商品ID相同，这样的列也会删掉，所以我们通过对用户ID也上提一行来过滤。最后便得到了用户访问商品的入度，而如果按用户groupby的话就可以得到用户的入度特征了。当然这个商品图模型可以扩展出更多的特征，出于时间我们就没做下去了。

此外用户的有效交互时间表达的是用户停留在某商品的时间，比如用户对于一件商品可能是误点，点入后马上点出，这种情况就不能算作有效交互，通过阈值可以筛选出有效的交互时间，我们取的阈值是180秒。同样也可以像入度特征类似计算。

##### 商品特征
同样商品的特征也包含两部分，一部分是商品本身的属性特征，例如商品的评论、差评率、one-hot后的属性特征，另一部分是交互产生的商品及商品所属品牌的特征。更用户交互产生的特征类似，有商品的流量、该商品时间窗的销量/该商品同品牌商品销量（表达商品在品牌下热度，当然可以有品牌的热度）、用户对品牌操作/用户总操作（表达用户对品牌的偏好）等 。

##### 行为特征
行为特征主要体现在用户与商品的交互，例如时间窗口内关注转加购率等一些转化率、用户对某件商品的入度、加购的商品数占购物车商品总数比、用户对商品操作的一些统计特征乘上时间衰减、用户对商品的有效交互、用户最后一次操作该商品距预测期时间等。

#### 模型训练
我们构建了两个模型，这两个模型都是用的xgboost，分别得到会产生购买的用户和购买的用户商品对。在模型训练方面我们通过交叉验证调节xgboost模型的参数，并通过5折交叉验证来得到最佳的迭代次数。

#### 结果融合
在得到这两个模型预测的用户和用户商品对后我们将其融合。融合是通过取top500然后求并集（例如预测的用户是A B C，预测的用户商品对是A-1 B-2 D-1，这个C用户没有出现在预测的用户商品对top500里面，我们就去找C用户最可能购买的商品然后添加进来），最后得到约700条用户商品对数据作为提交结果。

#### 代码地址
https://github.com/ScNico/JData-algorithm-competition