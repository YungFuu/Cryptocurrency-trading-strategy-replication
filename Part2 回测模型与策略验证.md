在Part 1 中，我们已将数据全部采集完毕，接下来要做的，是搭建一套回测的系统。结合论文，制定量化交易策略的整个项目的基本思路是：

1. 筛选标的

2. 根据因子判断多空

3. 执行多空策略，记录持仓状况

4. 计算收益，评估超额收益等指标，验证策略

**回测可以理解为是一个时间序列的遍历**，按照时间顺序，依次执行上述的前三点，即选股，根据因子判断多空，执行多空策略，记录持仓状况，这样一来，整体的思路就打开了。值得注意的是，在量化交易过程中经常会忽视的一个问题，即真实情况下，数据在某一时刻可能并不可获得，**我们必须要基于过往信息，对当下情况进行判断并执行交易。在这里，我事先约定以t-1的信息，在t时刻进行判断和交易。**

根据以上思路，我们可以先建立一个回测的框架。

```python

portfolio = pd.DataFrame()
long=pd.Series(dtype=float)
short=pd.Series(dtype=float)


for week in tqdm(range(min(df_week['week'])+1,max(df_week['week'])+1)):
    data = crypto_select(df_week,week) #在这一步筛选出股票/加密货币
    #To Decide what stock to long or short
    long,short = make_strategy_by_size(data,1) #在这一步按照选择的因子对加密货币进行多空的判断
    #Execute the strategy and save postion in "portfolio"
    execute_strategy(long,short,week) #执行上述判断，将收益记录在组合中


portfolio = portfolio.dropna()
PRC_result = cpt_return(portfolio) #计算收益
print(PRC_result)
PRC_result = cpt_strategy_ret(PRC_result) #计算策略收益
PRC_result.to_csv(path+os.sep+'PRC_result.csv',encoding = 'utf_8_sig')
runregression(PRC_result)  #评估超额收益


```

# 筛选标的

在原论文中，作者仅选取了所有市值超过100万美元的加密货币，这类加密货币一般发行量较大，价值较高，流动性更好，价值也相比许多不知名的小种类加密货币稳定。实现这一需求很简单，将满足要求的加密货币筛选出来即可。

```python
def crypto_select(df,week):
    #According to the info of last month return data.
    #In other words, we use T-1 info and do transaction at T
    last_week = week-1
    data = df[df['week']==last_week]
    data = df[df['mcap']>1000000]
    return data
```

这一步实现了两个需求，一个需求是基于历史可获取的信息来进行后续的判断，另一需求是筛选出符合标准的加密货币。

# 根据因子判断多空

在原论文中，作者根据加密货币的特征表现，将池子里的加密货币划分为5个分位数，分别测试持有1至5分位数中加密货币的回报率，同时也测试了多头5分位数，空头1分位数的加密货币的回报率，即5-1，从而得出多空策略。拆解上述需求，即需要我们实现两点，将加密货币按某一指标划分多个bins，并根据bins归类为long or short中。

这里仅以规模因子为例，论文中策略验证显著的是PRC，MAXDPRC，MCAP这三个因子，用pandas库的qcut函数划分为五个分位数，在根据分位数分类为long和short部分。这里不用纠结到底应该long指标高的还是long指标低的，只要根据结果收益率的正负来判断即可。具体代码实现路径如下：

```python
def make_strategy_by_size(data,n):  
    #Strategy: longs the smallest coins and shorts the largest coins generates
    if n == 1:
        data['PRC_bins'] = pd.qcut(data['PRC'], 5,labels = False)
        long = data[data['PRC_bins'] == 4]['crypto']
        short = data[data['PRC_bins']== 0]['crypto']
    elif n == 2:
        data['MAXDPRC_bins'] = pd.qcut(data['MAXDPRC'], 5,labels = False)
        long = data[data['MAXDPRC_bins'] == 4]['crypto']
        short = data[data['MAXDPRC_bins']== 0]['crypto']
    elif n==3:
        data['AGE_bins'] = pd.qcut(data['AGE'], 5,labels = False)
        long = data[data['AGE_bins'] == 4]['crypto']
        short = data[data['AGE_bins']== 0]['crypto']
    elif n==4:
        data['MCAP_bins'] = pd.qcut(data['MCAP'], 5,labels = False)
        long = data[data['MCAP_bins'] == 4]['crypto']
        short = data[data['MCAP_bins']== 0]['crypto']
    return long,short
```

代码执行完毕后，long，short分别以dataframe的形式记录了多头的加密货币信息和空头的加密货币信息。

# 执行多空策略，记录持仓状况

与其说是执行，不如说是按照多空的加密货币进行筛选，最后将信息存到portfolio里用于后续收益的计算。这个portfolio就记录了我们一共进行了哪些交易，或者换句话说，只保留原有数据库中进行过交易的加密货币信息，另外新增一列表明position是long还是short。具体代码实现路径如下：

```python
def execute_strategy(long,short,week):
    global portfolio
    long_portfolio = pd.DataFrame()
    short_portfolio = pd.DataFrame()    
    dataset = df_week[df_week['week'] == week]
 
    if long.empty:
        pass
    else:
        long_portfolio=dataset[dataset['crypto'].isin(long)]
        long_portfolio['position'] = 'long'
            
    if short.empty:
        pass
    else:
        short_portfolio =dataset[dataset['crypto'].isin(short)]
        short_portfolio['position'] = 'short'
    
    portfolio = pd.concat([long_portfolio,portfolio],ignore_index=True)
    portfolio = pd.concat([short_portfolio,portfolio],ignore_index=True)
```

# 计算收益

这一部分是分别计算多头与空头的收益率，ew指的是equal weighted，而vw指的的value weighted,后者将收益率按照市值进行了加权。

```python
def cpt_return(x):
    long = x[x['position'] == 'long']
    short = x[x['position'] == 'short']

    long_portfolios = (
        long
        .groupby(['week','position'])
        .apply(
            lambda g: pd.Series({
                'portfolio_ew_long': g['week_ret'].mean(),
                'portfolio_vw_long': (g['week_ret'] * g['mcap_lag1']).sum() / g['mcap_lag1'].sum()
            })
        )
    ).reset_index()

    
    short_portfolios = (
        short
        .groupby(['week','position'])
        .apply(
            lambda g: pd.Series({
                'portfolio_ew_short': g['week_ret'].mean(),
                'portfolio_vw_short': (g['week_ret'] * g['mcap_lag1']).sum() / g['mcap_lag1'].sum()
            })
        )
    ).reset_index()

    portfolios = pd.merge(long_portfolios,short_portfolios,on='week',how='outer')
    return portfolios        
```

portfolios返回的结果，即是week，position以及return这三个字段，记录从第一周到最后一周的多头总计收益与空头的总计收益。

# 计算策略收益

根据多空头的收益率，便计算出整个策略的收益率（即多头收益率-空头收益率），再将收益率进行累乘计算累计收益率，再将结果以图表的形式进行输出。

```python
def cpt_strategy_ret(df):
    df = df.replace(np.nan,0)
    df['strategy_vw'] = df['portfolio_vw_long'] - df['portfolio_vw_short']
    df['strategy_ew'] = df['portfolio_ew_long'] - df['portfolio_ew_short']
    
    plt.rcParams["figure.figsize"] = (10,7)
    df = df.sort_values('week') 
    df['cum_ew'] = (df['strategy_ew'] + 1).cumprod() - 1 
    df['cum_vw'] = (df['strategy_vw'] + 1).cumprod() - 1 
    
    draw_curve_cum_ret(df)
    draw_curve_ret(df)
    
    print('avg_long_ew: ', np.average(df['portfolio_ew_long']))
    print('avg_short_ew: ',np.average(list(map(lambda x: -x,df['portfolio_ew_short']))))
    print('avg_long_vw: ', np.average(df['portfolio_vw_long']))
    print('avg_short_vw: ',np.average(list(map(lambda x: -x,df['portfolio_vw_short']))))
    return df

def draw_curve_cum_ret(X):
###make a plot of roc curve
    plt.figure(dpi=150)
    lw = 2
    plt.ylim((round(min(X['cum_vw']),0),round(max(X['cum_vw']),0)))
    plt.plot(X['week'], X['cum_vw'], color='navy',lw=lw,label='cum_vw')
    plt.xlabel('Week')
    plt.ylabel('Cumulative Return')
    #plt.savefig(path+os.sep+save_name+'.jpg')
    plt.show()
    #print('Figure was saved to ' + path)
    return plt.show()

def draw_curve_ret(X):
    plt.figure(dpi=150)
    lw = 2
    plt.ylim((round(min(X['strategy_vw']),0),round(max(X['strategy_vw']),0)))    
    plt.plot(X['week'], X['strategy_vw'], color='navy',lw=lw, label='strategy_vw')
    plt.xlabel('Week')
    plt.ylabel('Return')
    #plt.savefig(path+os.sep+save_name+'.jpg')
    plt.show()
    #print('Figure was saved to ' + path)
    return plt.show()
```



输出结果：

avg_long_ew:  2.9774913128295274e+39
avg_short_ew:  -16.330047115866513
avg_long_vw:  0.015973559178956997
avg_short_vw:  -0.039559329281993544

![](C:\Users\pc\AppData\Roaming\marktext\images\2022-05-30-16-53-46-image.png)

从下图可看出累计收益率是负数，但别忘了我们这是在制定量化交易策略，如果将多空头调换过来，那么累计收益率将在约200周后超过100%。从平均收益来看，多头的周收益率为1.5%，空头的周一率为-3.9%，综上来看，应当采取long 低PRC的加密货币，short高PRC的加密货币这样的策略，即实际应采取的策略为1-5而非5-1。另外，以上收益率为周的回报率，若换算为年回报率，那数字将不可小觑。

不过究竟是这一策略所带来如此丰厚的收益，还是因为其他市场共性因素导致的，则还需要在后续的单因子模型中加以确认。

# 加密货币单因子模型

原论文中采用了加密货币单因子模型来验证其成功的10种加密货币多空策略，该模型与CAPM模型比较相似，CMKT指的是加密货币的超额市场收益，具体如下所示，

<img src="file:///C:/Users/pc/AppData/Roaming/marktext/images/2022-05-30-17-03-12-image.png" title="" alt="" width="402">

在之前的数据清洗与处理工作中，CMKT的数据我们已计算得到，而在先前的代码中，我们得到了策略收益Ri，现在，只需要运用这些数据跑一遍线性回归模型即可。

```python
def runregression(result):
    result = pd.merge(result,df_week[['week','week_Rf', 'CMKT']].drop_duplicates(),on = 'week')
    result['strategy_vw_rf'] = result['strategy_vw'] - result['week_Rf']    
    print(smf.ols('strategy_vw_rf ~ 1 + CMKT', result).fit().summary())
```

输出结果：

```python
                            OLS Regression Results                            
==============================================================================
Dep. Variable:         strategy_vw_rf   R-squared:                       0.000
Model:                            OLS   Adj. R-squared:                 -0.002
Method:                 Least Squares   F-statistic:                    0.1589
Date:                Mon, 30 May 2022   Prob (F-statistic):              0.690
Time:                        16:59:32   Log-Likelihood:                -37.184
No. Observations:                 411   AIC:                             78.37
Df Residuals:                     409   BIC:                             86.40
Df Model:                           1                                         
Covariance Type:            nonrobust                                         
==============================================================================
                 coef    std err          t      P>|t|      [0.025      0.975]
------------------------------------------------------------------------------
Intercept     -0.0230      0.013     -1.733      0.084      -0.049       0.003
CMKT          -0.0486      0.122     -0.399      0.690      -0.289       0.191
==============================================================================
Omnibus:                      712.379   Durbin-Watson:                   1.797
Prob(Omnibus):                  0.000   Jarque-Bera (JB):           366540.191
Skew:                         -10.190   Prob(JB):                         0.00
Kurtosis:                     147.874   Cond. No.                         9.32
==============================================================================

Notes:
[1] Standard Errors assume that the covariance matrix of the errors is correctly specified.
```

与原论文的情况比较相似，取得的R2并不是很高，但截距项的t检验还是比较显著的，在90%的显著性水平下，可以认为该策略能够带来超额收益。

针对不同的因子重复进行上述的代码操作，最后，取得的结果如下：

**![](https://lh3.googleusercontent.com/3fjRqW3wUrL5o_gI0p0ZTK1_WsSJVUoeDPTMMxUPvFm7N5IMNYkB9AhyLOLSRB2iCACDT5Hn4mlNgkhmER9JYhlC6Jhh1jT7VIP7KY4OnBFK_74dF8SfqN-irMPuF56CgLuHNrQ)**

从结果上看，基于规模因子，动量因子和交易量因子这三种类型的因子的交易策略都能带来不同程度的超额收益。

以下是原论文的结果：

![](C:\Users\pc\AppData\Roaming\marktext\images\2022-05-30-17-16-57-image.png)

原论文的结果与我的结果还是存在较大程度的差异，差异的原因可能来源于数据集的不同，选取的时间范围不同，而这也是我首次涉猎量化方面的代码，可能多少存在不少疏漏之处与处理欠妥当的地方，本项目仍有许多改进的空间，欢迎各位提出改进建议，友好地进行交流，如有问题，我也将尽我所能进行回答。










