## 零、正式研究流程

​	这里要写明白每一部分需要进行的操作应该具体去哪个文件里面找

### 1、尝试构建alpha

​	根据已知的因子进一步思考，尝试进一步找到更强的信号。

​	首先思考这个因子根本的原理是什么，即背后从金融上可解释的思路，然后基于思路，对于每一个金融意义的值采用不同的实现方式（不同的参数），观察效果如何。

- 要仔细想公式背后的逻辑是什么，这是获得更好的表现的方式
- 停下来深入思考比不停尝试公式更加重要
- 要知道自己的尝试背后是出于什么样的原因

### 2、尝试测试

​	最终测试一个alpha的顺序： **需要明白在每一步中什么样子的表现是好的**

- **每次回测：** 打印superRun输出。良好的纸面表现是起点。

- **小的改动，如更改操作参数：** 检查niub_new结果是否有所改善（和已经有了的因子之间的相关系数）

- **寻找变化方向：** 可以在不同的自编因子上检查multibcorr（和自己的因子之间的相关系数，尝试组合）

- **中等改动，如添加一个操作：** 检查交易约束、高流动性uv和高百分位表现

- **稳定版本：** 检查vatest.py，就是5.2里面上传到服务器上面的方式

> 一些应该到时候整理进去的备注：
>
> - 要注意检查alpha在高流动性股票上的表现
> - alpha的turnover应该比较低，一般需要低于40%

- 用哪些变量进行中性化处理需要自己去尝试寻找，多了会额外引入噪声，这是一个需要自己尝试的方向

如果想要查看数据的话，应该直接print，就在python的脚本里面，从文件夹翻是比较费劲的方式

## 一、Pysim框架

### 1、基本概念

​	alpha衡量投资组合相对市场的return表现，beta衡量投资组合相对市场的risk（std）表现，市场的alpha是0，beta是1。

​	除了标准差之外，还有一些对于risk的衡量方式。比如downside risk，表示投资在最坏情况下可能的损失。我们使用semi-variance（计算所有低于平均值的点的方差）对于downside risk进行度量。

​	IC（information coefficient）是预测的return和实际的return的相关系数，IR（information ratio）是每日盈亏（daily pnl）的平均值除以标准差（归一化后的平均值）。IC反映预测的准确性，IR反映单位风险获得的超额回报。存在公式IR = IC * sqrt（breadth），breadth是决策的数量或者频率，说明增加超额回报可以通过增加预测数量和增加预测准确性这两个方面来实现。

​	一般来说预测股票价格的能力比较弱，IC一般是0.01 - 0.03左右，这已经算比较好的了。

​	我们可以使用多空策略（long-short strategy）来规避掉市场的影响，使用一半资金做多、一半资金做空。这种行为称为市场中性化(market neutralization)，相当于把市场的alpha从股票中减掉了，类似于一种控制变量的思想。除了市场之外，也可以对因子进行中性化。因为希望找到能独自影响的因子，而不是靠别的因素影响的。

​	中性化就是先设定一些风险因子，然后做线性回归之后取残差，得到和这些风险因子线性无关的部分。

​	在这里，alpha指有统计显著性的盈利预测模型，基本的分类为：

- 风险因子 & 阿尔法因子
  - 风险因子是risk不是alpha，是基本上公认且无法预测的，比如公司规模和行业
    - 有的时候大公司普遍表现好，有的时候小公司表现好，无法确定
    - 有的时候这个行业表现好，有的时候这个行业表现差，同样无法确定
    - 阿尔法因子是可以用来预测并获得收益的，就像上面定义的那样
- 阿尔法因子可进一步分类
  - 量价因子：历史数据驱动的
  - 基本面因子：比较长时间的非价格信息，比如估值、成长性和盈利质量
  - 事件驱动因子：比较短期的非价格信息，比如财报发布、突发事件等
  - 另类数据因子

### 2、回测框架

​	这部分说明如何使用pysim回测框架进行操作，主要关注如何运行文件以及阅读回测的结果。

​	使用`./superRun.py config_mini.xml`格式的命令进行回测，回测过程中的结果会存储在pnl文件夹中。最终回测结果中每一列的含义如下，衡量的是具体的某个因子在这段时间中的表现：

```
•	dates: 模拟日期
•	long: 分配给多头头寸的规模（以百万计）
•	short: 分配给空头头寸的规模（以百万计）
•	pnl: 头寸和交易产生的资金（以百万计）
•	%ret: 回报，资本使用回报
•	%tvr: 换手率，定义为交易股数/持有股数
•	shrp: 夏普比率，或信息比率（IR），定义为回报的平均值/回报的标准差。IR = 夏普比率 / sqrt(一年中的交易天数)
•	dd: 期间的最大回撤
•	%win: 盈利天数的百分比
•	margin: 定义为pnl / 交易股数
•	fitness/fitsc: 定义为夏普比率 * sqrt(回报 / 换手率)。较高的夏普比率、较高的回报、较低的换手率会得到较高的适应度得分。
•	lnum: 多头头寸中的平均股票数量
•	snum: 空头头寸中的平均股票数量
•	tdays: 持有股票数量>0的天数
•	tratio: tdays / 总交易天数的比率
```

​	在回测过程中的数据，衡量的是某个因子在某一天中的表现，具体每一列对应的含义是：

```
date     alpha_id    long x short    long_stocks x short_stocks trade_shares daily_pnl 
20220601  alpha01 9999999 x -9998245        2537 x 2059              1319540     39764  

cumulative_pnl dummy_col1 dummy_col2 turnover avg_return max_drawdown wt-adjust    ir      
2784764          0      0.000   0.479       0.119       -0.133      1.00 0.057
```

​	在pnl中存储的文件的内容对应格式如下：

```	
daily pnl: Date PNL Long Short Return Holdvalue Tradevalue Holdshares Tradedshares Longcount Shortcount
intra pnl: Date PNL Long Short Return Holdvalue Tradevalue Holdshares Tradedshares Longcount Shortcount Holdpnl
```

​	这些字段具体对应的值是：

```
•	Date: 模拟日期
•	PNL: 头寸和交易产生的资金
•	Long: 分配给多头的规模
•	Short: 分配给空头的规模
•	Return: 资本使用回报
•	Holdvalue: 规模
•	Tradevalue: 用于实现策略的资金量
•	Holdshares: 持有的股票总数
•	Tradedshares: 当日交易的股票总数
•	Longcount: 将持有多头头寸的股票数量
•	Shortcount: 将持有空头头寸的股票数量
•	Holdpnl: 如果保持昨天的头寸，pnl
```

​	我们可以使用已经写好的simsummary.py和simsummary_intra.py这两个脚本来对pnl文件进行总结，具体操作为：

```python
./tools/simsummary.py ${pnl_file}
./tools/simsummary_intra.py ${intraday_pnl_file}
```

​	生成的结果中之前没有解释的行的含义是：

```
•	turnover: 定义为tradedshares / holdshares
•	dd: 期间的最大回撤
•	margin: 定义为pnl / tradedshares
•	imargin: 定义为intraday_pnl / tradedshares
•	fitness/fitsc: 定义为夏普比率 * sqrt(回报 / 换手率)。较高的夏普比率、较高的回报、较低的换手率会得到较高的适应度得分。
•	tdays: 持有股票数量>0的天数
•	tratio: tdays / 总交易天数的比率
```

​	粗略地讲，一个比较好的alpha应该满足如下的要求：

```
•	合适的IR (> 0.1)
•	与回报相匹配的回撤（dd < return / 3）
•	与pnl相匹配的换手率（margin > 5）
•	每天有接近1 * 10^7的多头和空头
•	每天多头和空头股票数量大致相同（不适用于事件alpha）
•	tratio非常接近1（不适用于事件alpha）
```

## 二、代码文件解释

​	有两种描述一个alpha的方式，使用xml文件或者使用python文件。

### 1、xml文件

​	xml文件是回测脚本的输入，每一个xml文件唯一对应一个alpha，包含这个alpha的所有配置。实际操作的时候也要让xml和alpha一一对应。

​	这一部分主要介绍xml文件是如何描述一个alpha的。**但是很多都没有被解释，说之后会具体解释**

```	xml
<?xml version='1.0'?> 
<PySim xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="pysim" xsi:schemaLocation="pysim /dat/pysimrelease/pysim-5.0.0/pysim.xsd">
上面的这两行是alpha和回测系统的路径，不能修改
  <Macros MODULES="/dat/pysimrelease/pysim-5.0.0/build/lib.linux-x86_64-3.8/pysim/modules" EXAMPLES="/dat/pysimrelease/pysim-5.0.0/examples" PYEXTPATH="/dat/pysimrelease/pyext-3.0.0" EXPRLIB="/dat/pysimrelease/pyalphaexpr3"/> 这是用来指定需要调用的文件的相对路径的
  <Consts startdate="20200101" enddate="TODAY" backdays="20" cachepath="/dat/cqcache/data" __checkpointDir="./checkpoint" __checkpointDays="5"/> 这里设定了在什么时间段测试，实际操作中最好让startdate相同，这样得到的结果是可以相互比较的；checkpoint会存储回测的内容并在之后使用，在测试中我们会希望测试新的参数，并不想把数据存下来，因此使用__把这个功能禁用了
  
  <!-- datasets --> 这一部分负责加载数据
  <Datasets> 
    <!-- universe -->
    <Data id="ALL Ashare TOP100 TOP150 TOP450 TOP300 TOP1000 HS300 ZZ500 TOP2000"/>
    <!-- data(s) -->
    <Data id="Fxrate ISO BaseData CAX slippage"/> 需要把会用到的数据在这里添加进来
    <Data id="Returns Returns5 Returns20 Returns50 Returns120 Returns250 Returns1 Returns3"/>
    <Data id="WindIncome WindIncomeMatrix"/> 这里是加载了的数据的id，是/dat/cqcache/data文件夹下的
    <Data id="TradeStatus"/> 例如想使用country_origin.N,6656I，就要在这里添加country_origin
    <!-- special data(s) -->
    <!-- intraday data-->
    <Data id="IntervalData" path="${MODULES}/DataLoadcache.so" intraday="true" intervalSteps="67"/>
    <Data id="Intervalstats" path="${MODULES}/DataLoadcache.so" intraday="true" intervalSteps="49"/>
    <!-- in-memory data -->
    <Data id="AdjPrices" path="${MODULES}/DataAdjprices.so"/>
  </Datasets>
  
  <!--modules including: Alphas, Operations, combos, calculations, portfolios -->
  <Modules> 这一部分说明调用哪些地方的代码文件进行回测操作，这里指定的文件会负责生成alpha
    <!-- Alphas -->这里的id是自己起的，path就是代码文件的位置
    <AlphaModule id="AlphaExampleMini" path="./AlphaExampleMini.py"/> 这是第一个alpha，对应一个python脚本
    <AlphaModule id="ExprLoader" path="${MODULES}/ExprLoader.so"/>	这是第二个alpha，应该是对应后面的表达式，不过path中的MODULES是什么意思
    <!-- Operation modules -->
    <!-- offical operators, no need to set path, don't change name  -->
    <OpModule id="OpPow OpGroupNeut OpDecay OpRiskNeut OpEMADecay OpHump OpNorm OpTruncate OpTradeConstraint OpVectorNeut OpWinsorize OpGroupNeutFast OpNio"/> 这是实现好的脚本，可以调用，在model文件夹中，每个脚本中都有解释的注释，可以进行归一化、乘方等操作
    <!-- customer op, need set path -->
    <OpModule id="OpPower2" path="${MODULES}/OpPow.so"/>
    <OpModule id="OpDecay2" path="${EXAMPLES}/OpDecay.py"/>
    <!-- Combo modules -->
    <ComboModule id="ComboSimple" path="${MODULES}/ComboSimple.so"/>
    <!-- Calculation modules -->
    <CalcModule id="CalcSimple" path="${MODULES}/CalcSimple.so"/>
    <CalcModule id="CalcIndex" path="${MODULES}/CalcIndex.so"/>
    <!-- Portfolio modules -->
    <PortModule id="PortSimple" path="${MODULES}/PortSimple.so"/>
    <PortModule id="PortExprSimple" path="${MODULES}/PortExprSimple.so"/>
  </Modules>

  <Port size="20e6" mId="PortSimple" id="MyPort" homecurrency="CNY" comboId="ComboSimple"> 这里构建alpha
    <Calc mId="CalcSimple" pnlDir="pnl"/> 这行和上面的那一行不需要修改
    <Alpha mId="AlphaExampleMini" id="alpha01" uId="Ashare" size="20e6" delay="1" days="10"> 这是第一个alpha
      上面的mId要和AlphaModule中的id一致，id是回测框架中的名称（要独特），uId连接到Data模块中加载的数据，说将来会解释这些数据是什么调用情况；size是规模，一般不需要修改；delay表示延迟几天，通常保持为1，但可以变大来测试alpha的鲁棒性
      <AlphaInfo name="alphadummy" author="TianTian" category="price_volume" instrument="equity" region="chn" intervalmode="daily"/>是描述alpha的信息，研究的时候可以删掉，只需要在提交的时候加上就可以
      <Op mId="OpPow"/> 这里表示如何对alpha的结果进行操作，mId对应OpModule中的那些Op开头的模型
      <Op mId="OpGroupNeut" group="industry"/> 
    </Alpha>
    <Alpha mId="ExprLoader" id="alpha02" uId="Ashare" size="20e6" delay="1" exprname="a0"> 这是第二个alpha，mId对应的模块会从下面的PortExpr模块中加载算出来的结果，exprname会和下面的Expr中的name对应
      这里的alpha就是下面expression算出来的结果
      <AlphaInfo name="alphadummy" author="your_name" category="price_volume" instrument="equity" region="chn" intervalmode="daily"/> 基本的备注信息
      <Op mId="OpPow"/> 
      <Op mId="OpGroupNeut" group="industry"/> 看起来像是对于行业进行中性化的操作
    </Alpha>
  </Port>
  
  <PortExpr mId="PortExprSimple" id="MyPort3" pnlDir="./pnl" uId="Ashare" >这里表示如何计算表达式（中间变量），比如a0就是对应expression计算的结果，下面的这个表达式意思是计算过去十天的return的平均值
    <Expr name="a0" expression="-tsmean(returns, 10)" doScale="true" doStats="false" delay="1" uId="Ashare"/>
  </PortExpr> 这里就是第四部分讲的可以用来快速计算的表达式
</PySim>

```

### 2、python文件

​	这里解释的python代码是上面xml文件中第一个alpha调用的，和上面的第二个alpha的表达式相同。

```python
import framework.Universe as uv
from framework.Alpha import AlphaBase
from framework.Niodata import *
# 上面这三行导入pysim框架，IDE报错是正常的
import numpy as np
class AlphaExample(AlphaBase): # 这个类是用来算每天的alpha值的，得到的数据都是对于某一天的
    def __init__(self, cfg): # cfg是从配置文件中读取的数据，配置文件就是xml文件
        AlphaBase.__init__(self, cfg)
        self.days = cfg.getAttributeDefault('days', 10) # 这是读取参数的方式，前一项key对应属性的名字，后一项提示想要的数据是什么类型的（这里是整数类型），如果输入的不是10是字符串，会返回一个字符串类型的日期
        # 如果想要的参数比较复杂，比如是列表，可以用下面的这种方式读取list_parameter作为key对应的数据1,2,3
        # list_parameter = cfg.getAttributeDefault('list_parameter', '').split(',') # 即把返回的数据用','分隔开
				# self.list_parameter = [int(x) for x in list_parameter]  # 然后依次读取、类型转换返回的数据并存储在列表中
        self.ret = self.dr.GetData('returns') # 这里直接根据名称加载想要的参数的名字
        self.close = self.dr.GetData('close') # 这里也是，得到的值可以当作numpy的ndarray
        self.industry = self.dr.GetData('industry')
    def GenAlpha(self, di, ti=None): # 这里具体计算某一天alpha的数值
      	# di是天的迭代索引，uv.Dates[di]可以得到实际表示日期的整数，ti用来索引日内的时间
        self.alpha = -np.nanmean(self.ret[di-self.delay-self.days+1:di-self.delay+1, 0:uv.instsz], axis=0)
        # self.delay是从AlphaBase继承的，即xml中Alpha模块定义的，一般来说是1
        # self.ret[过去十天的切片, 股票索引的长度(un.instsz是一个确定比A股中所有股票都多的常数)]
				# 因为股票索引比较大并且很多股票退市，可能有很多NaN，使用np.nanmean可以在计算中忽略NaN值
        # print(np.sum(np.isnan(self.ret[di-self.delay])) / uv.instsz)可以计算出来每天的return数据中NaN占比
        # nioIsvalid（从framework.Niodata导入的函数)可以类似地和isnan（对于浮点数）一样，检测int数据对应的NaN的值
        # 如果不添加axis，会把二位的数组展平成一维计算平均值，axis为0时是对时间维度计算，为1是计算的是横截面的平均值
        self.alpha[self.valid[di, 0:uv.instsz] != 1] = np.nan
        # self.valid[di, 0:uv.instsz]对应在di这一天所有股票是否有数据，对于没有的要确保是NaN，这行代码可以不修改
def create(cfg): # 会被pysim平台调用，返回的AlphaExample会被用来计算每天的alpha
    return AlphaExample(cfg)
```

​	用python文件构建alpha的话不需要具体进行中性化，因为python文件是被xml文件调用的，调用的时候会完成风险中性化的设定。

## 三、数据解释

​	数据存储在缓存cqche中，这样可以读取起来更快。

​	数据类型有：MATRIX（正常的二维数据），VECTOR（一维的事件数据），CUBE（三维的，如盘中数据），IX（字符串）。

​	可以用`ls {被搜索路径} | grep {搜索词} -i`来查找需要的数据，`-i`是用来忽略大小写的，这个命令只会在当前目录下查找，不会递归进行搜索。

​	如果想进行分类操作，例如按照行业对股票分组，可以使用IX数据，具体在文件中。	

​	`BaseData`是主要使用的数据，其中一些可能比较重要的数据如下：

```
	•	ticker：ii 对应的股票代码。股票代码看起来像 {6 位数字}.{SS 或 SZ 或 BJ}。SS 是在上交所交易的股票，SZ 是深交所交易的股票，BJ 是北交所交易的股票。外部数据通常使用 SH 表示上交所，你应该在任何比较前转换它。你也可以在任何搜索引擎或股票信息网站上搜索 6 位数字以获取股票信息。
	•	close：交易结束后的股票价格。最具代表性的一天股票价格。
	•	sharesout：公司的流通股数量
	•	cap：公司的市值 = close * sharesout
	•	volume：一天内交易的股票数量
	•	vwap：成交量加权股票价格。代表一天内股票的“平均”交易价格
	•	open：集合竞价开始时的股票价格。与昨天的收盘价不同。我们称今天开盘/昨天收盘 - 1 为隔夜收益率。它也有一些意义。
	•	high, low：一天内的最高和最低价格
	•	Returns/returns：股票的日回报 = close / 昨天 close - 1。
	•	ReturnsXXX/returnsxxx：XXX 天的回报
	•	CAX/split：拆分因子，由于转股/送股增加的 sharesout，这将理论上导致股票价格的相反变化并保持 cap 不变。因此，昨天的价格只能与今天的价格/今天的拆分进行比较。
	•	WindIndustry/WindIndustry.wind1 和 wind234：万得的行业分类。通常比 BaseData/industry 更优选。4 有最深入的分类，最多的行业和每个行业中最少的公司数。你可以从使用 Wind2 开始。
```

​	BasicFundamentalRatio 包含一些常用的价格与基本面比率，OR 代表营业收入，OP 代表营业利润，GR 代表毛收入。

​	建议先从流动性比较好的开始研究，例如TOPLIQ65。

​	这种内存数据只能在python文件中使用，不能直接在alpha expression中调用。

## 四、表达式快速计算

​	如何使用表达式快速测试alpha，以及一些具体可以提升alpha性能的方式。

​	具体的要去看pdf里面的具体代码案例，有具体可以调用的函数和具体怎么操作。

​	可以直接在xml文件中写出来表达式来计算，也可以在xml文件中调用python文件来计算（就是用自己之前写出来的现成的因子），具体实现细节在pdf中，和实现alpha的脚本类似，可以在脚本中直接调用别的alpha来配置。可以把脚本存储在磁盘来加速配置的速度，具体怎么做看文档，但是要记得正式提交的时候删掉。

​	然后讲了一下具体对alpha的判断，把具体讲的怎么判断放到下面评估的部分里了。

## 五、alpha相关

### 1、如何寻找

​	根据已知的因子进一步思考，尝试进一步找到更强的信号。

​	首先思考这个因子根本的原理是什么，即背后从金融上可解释的思路，然后基于思路，对于每一个金融意义的值采用不同的实现方式（不同的参数），观察效果如何。

- 要仔细想公式背后的逻辑是什么，这是获得更好的表现的方式
- 停下来深入思考比不停尝试公式更加重要
- 要知道自己的尝试背后是出于什么样的原因

​	具体的例子在第四部分的文件夹的txt文件中，具体展示了每一部分可以怎么调整参数。

​	进行中性化的时候，使用因子的rank比使用实际的值更好，可以规避掉离群值的影响。

​	只有当两个alpha来自相同的idea但实现的参数不同时，才可以尝试组合。

​	第四部分的txt文件中具体讲解了如何进行组合操作。

### 2、如何评估

- 如何检验两个alpha之间的相关性

​	可以研究两个alpha回测得到的pnl的相关性来看看表现如何，指令为`/dat/pysimrelease/pysim-5.0.0/tools/multibcorr pnl/alpha01 pnl/alpha02 pnl/alpha03 2 2`，这是对三个alpha进行相关性检验。

- 如何选择alpha的组合  **疑问：之前不是说最好不要组合吗，还是说不要和已经有了的alpha进行组合**

​	把候选的alpha作为y，对确定的alpha进行线性组合，可以通过残差项来衡量增加这个alpha可以带来的额外效果，残差项的大小被称为增值value added。

- 检验构建的alpha和已有的alpha的相关性

​	这里检验的是pnl的相关性，可以直接调用已经有了的库来实现：`/dat/pysimrelease/pysim-5.0.0/tools/niub_new {alpha pnl 文件路径}`。

​	得到的结果中分割线上面是相关性最高和最低（由于有正负号，其实就是相关性的绝对值最高的）的各10个alpha，分割线下面是另一种“增值”最低的10个alpha，但这里计算的“增值”越大越糟糕。

​	如果出现了非常高相关性的alpha，应该咨询mentor是为什么。

​	我们希望得到最大的相关系数 < 0.4 且最大的“增值” < 2的alpha，< 1是最理想的情况。

- 检查和变量的相关性

​	直接检查alpha和数据或者构建的别的alpha的相关系数，如果高的话应该从金融层面思考是为什么，然后改为使用无关的数据计算参数值，例如把数值型变量变成它的rank排序结果。

​	在表达式中，可以用`vectorneut(a, b, demean=true)`得到和b线性无关的a的部分。

### 3、测试实际约束

- 针对short的限制

​	现实中不能只short一支股票，需要short一个index，**模拟是什么操作原理没太看懂**

- 针对流动性的

​	使用AlphaCopy进行测试，修改到别的测试集上面进行测试

- 涨跌停

​	可以修改OpModule中的值，来禁止在涨跌停了的股票上交易

​	如果alpha到了最终的检查部分，有一个比较费时间的可以最终检查alpha的方式，在5.2文件的Value Add Test部分。

​	还有统计层面的检验和画图的IC检验，这些都是和Value Add Test相关的，会上传到一个特定的网站上进行检查。

## 六、最终提交

​	到时候如果真的有能提交的再看。

- 实际运行的时候如何加上checkpoint来算的更快一些，提交之前需要修改config文件
- 正式提交alpha之前的流程，让代码更加的模块化，不要有硬编码的变量（不管是数值的还是字符串的）

## 七、附录

### 1、数据分析教程

- np和pd如何使用，以及一些实际的例子

### 2、向量数据

- 如何使用向量数据（不是按照天或者股票索引的）

### 3、checkpoint实现

- 如何具体存储checkpoint

### 4、FAQ



## 八、已经有了的代码文件





## 尚未解决的问题

