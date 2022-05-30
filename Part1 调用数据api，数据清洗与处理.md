# 写在开头

这篇文章是介绍我如何复现他人的学术论文，这也是我在HKU量化交易课程期末的项目。

你可以Google文章名或通过以下链接查阅该论文：LIU, Y., TSYVINSKI, A. and WU, X. (2022), Common Risk Factors in Cryptocurrency. The Journal of Finance, 77: 1133-1177. [Common Risk Factors in Cryptocurrency - LIU - 2022 - The Journal of Finance - Wiley Online Library](https://doi.org/10.1111/jofi.13119)

本期技术博客将分为两个部分来写，第一部分为获取数据，数据清洗与处理工作，第二部分将介绍如何实现交易策略制定，策略执行，回测模型搭建，并运用加密货币单因子模型，对策略所带来的超额收益进行估计。

# 关于论文

概况一下原论文的研究思路为：通过规模、动量、交易量以及波动性这四类主题因子，来分析不同特征下加密货币的市场表现，文章将加密货币每周的这四类特征，划分为五个分位数，并根据五分位数和一份位数之间的差异，形成多空策略。

关于主题因子具体有哪些，都有什么含义，可查看论文的Table Ⅱ，这里仅展示复现中所用到的因子。

| Category | Predictor | Reference                                                                                                                    | Definition                                                              |
| -------- | --------- | ---------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| Size     | MCAP      | Banz ([1981](https://onlinelibrary.wiley.com/doi/full/10.1111/jofi.13119#jofi13119-bib-0012))                                | Log last-day market capitalization in the portfolio formation week.     |
| Size     | PRC       | Miller and Scholes ([1982](https://onlinelibrary.wiley.com/doi/full/10.1111/jofi.13119#jofi13119-bib-0056))                  | Log last-day price in the portfolio formation week.                     |
| Size     | MAXDPRC   | George and Hwang ([2004](https://onlinelibrary.wiley.com/doi/full/10.1111/jofi.13119#jofi13119-bib-0038))                    | Maximum price of the portfolio formation week.                          |
| Mom      | r 1,0     | Jegadeesh and Titman ([1993](https://onlinelibrary.wiley.com/doi/full/10.1111/jofi.13119#jofi13119-bib-0047))                | Past one-week return.                                                   |
| Mom      | r 2,0     | Jegadeesh and Titman ([1993](https://onlinelibrary.wiley.com/doi/full/10.1111/jofi.13119#jofi13119-bib-0047))                | Past two-week return.                                                   |
| Mom      | r 3,0     | Jegadeesh and Titman ([1993](https://onlinelibrary.wiley.com/doi/full/10.1111/jofi.13119#jofi13119-bib-0047))                | Past three-week return.                                                 |
| Mom      | r 4,0     | Jegadeesh and Titman ([1993](https://onlinelibrary.wiley.com/doi/full/10.1111/jofi.13119#jofi13119-bib-0047))                | Past four-week return.                                                  |
| Mom      | r 4,1     | Jegadeesh and Titman ([1993](https://onlinelibrary.wiley.com/doi/full/10.1111/jofi.13119#jofi13119-bib-0047))                | Past one-to-four-week return.                                           |
| Volume   | VOL       | Chordia, Subrahmanyam, and Anshuman ([2001](https://onlinelibrary.wiley.com/doi/full/10.1111/jofi.13119#jofi13119-bib-0023)) | Log average daily volume in the portfolio formation week.               |
| Volume   | PRCVOL    | Chordia, Subrahmanyam, and Anshuman ([2001](https://onlinelibrary.wiley.com/doi/full/10.1111/jofi.13119#jofi13119-bib-0023)) | Log average daily volume times price in the portfolio formation week.   |
| Vol      | STDPRCVOL | Chordia, Subrahmanyam, and Anshuman ([2001](https://onlinelibrary.wiley.com/doi/full/10.1111/jofi.13119#jofi13119-bib-0023)) | Log standard deviation of price volume in the portfolio formation week. |

在得到多空策略以及不同特征下加密货币的市场表现后，再根据加密货币的“CAPM模型”，即可估计策略所带来的超额收益，关于加密货币单因子模型的介绍，可查阅论文的Table Ⅷ部分。

只要了解了原论文的研究思路，就很好对论文进行复现了。

# 关于数据

原论文通过调用[Coinmarketcap]([https://coinmarketcap.com/](https://coinmarketcap.com/))的api，获取了2014年年初至2020年7月所有**市值超过100万美元**的加密货币信息以及回报率，加密货币种类数量最多达到1559各个，想了解[Coinmarketcap.api](https://coinmarketcap.com/api/documentation/v1/#section/Authentication)的朋友可以通过链接查阅文档，我在这里不做介绍。

由于Coinmarketcap的API有较多限制，且需要付费（真的很贵）才允许使用，我也尝试了一些其他数据获取途径，比如investing.com中也有较多加密货币种类的数据，且可以点击下载，通过selenium写脚本也可以实现逐步下载，但由于其提供的数据字段有限，在初步尝试后我放弃了该网站，转而向另一个网站[CoinGecko](https://www.coingecko.com/zh)，这同样也是世界最大的独立加密货币数据汇总企业之一，跟踪了全球超过600+交易所中超过13000种不同的加密货币，其API提供有限制的免费访问途径，关于该API的文档可以可以点击这里[Crypto API Documentation | CoinGecko](https://www.coingecko.com/zh/api/documentation)。

# 获取数据

我们主要使用到CoinGecko的以下三条API：

```python
#连接CoinGecko的API，使用免费版本无需提供KEY
cg = CoinGeckoAPI()

#返回一个字典，其中为n个记录加密货币信息的子字典，子字典包含id，symbol，name三个字段。
cg.get_coins_list()

#coin_list = cg.get_coins_list()
# input：coin_list[1]
# output:{'id': '0-5x-long-algorand-token','symbol': 'algohalf','name': '0.5X Long Algorand'}

#获取指定加密货币一段时期内的数据，返回一个字典，字典的三个Key分别记录price，market_caps,以及total_volumes这几样信息。
#单个Key中，数据以列表形式存储，列表中包含时间戳及时间戳对应的数据信息。
cg.get_coin_market_chart_range_by_id(
        crypto,#加密货币id
        vs_currency='usd',#货币单位
        from_timestamp=begin_date,#起始日期
        to_timestamp=end_date)#终止日期

# input：info['prices'][0]
# output:[[1623196800000, 0.0014762190674748388],[1623283200000, 0.0017628434668555861]...]
#注意，from_timestamp 和 to_timestamp的两个参数所需要输入的信息是时间戳而非我们所常用的时间格式，需要进行一定的转换
```

获取数据的思路很简单，首先从CoinGecko拿到其拥有的所有加密货币的id，然后写一个便利函数，依次获取不同加密货币的数据即可，每拿到一个加密货币数据后，就存储到统一的DataFrame里，代码具体实现路径如下：

```python
def timeStamp(timeNum):
    # translate timeNum to standard time format
    timeStamp = float(timeNum/1000)
    timeArray = time.localtime(timeStamp)
    otherStyleTime = time.strftime("%Y-%m-%d %H:%M:%S", timeArray)
    return otherStyleTime

#Get all cryptocurrency name in the CoinGecko, save its id in id_list
cg = CoinGeckoAPI()
coin_list = cg.get_coins_list()
id_list =[]
for dic in coin_list:
    coin_id = dic['id']
    id_list.append(coin_id)

#Setup begin_date and end_date
begin_date = "2014-01-01 000000"
timeArray = time.strptime(begin_date, "%Y-%m-%d %H%M%S")
begin_date = int(time.mktime(timeArray))

end_date = "2021-12-31 000000"
timeArray = time.strptime(end_date, "%Y-%m-%d %H%M%S")
end_date =  int(time.mktime(timeArray))

df_total = pd.DataFrame()

#Get coin market data by id, save info in df_total
#Attention: For Free API, the request frequency is limited, so we wait 1 minute after every 50 requests
count = 0
# for crypto in tqdm(id_list[0:4481]):
for crypto in tqdm(id_list):
    df1 = pd.DataFrame()
    df2 = pd.DataFrame()
    df3 = pd.DataFrame()
    price_list=[]
    vol_list=[]
    mcap_list=[]
    
    info = cg.get_coin_market_chart_range_by_id(
        crypto,
        vs_currency='usd',
        from_timestamp=begin_date,
        to_timestamp=end_date)
    
    time_list=[]
    for row in info['prices']:
        timeNum = row[0]
        timeNum = timeStamp(timeNum)
        time_list.append(timeNum)
        price = row[1]
        price_list.append(price)
    df1['date'] = time_list
    df1['price'] = price_list
    
    time_list=[]
    for row in info['market_caps']:
        timeNum = row[0]
        timeNum = timeStamp(timeNum)
        time_list.append(timeNum)
        mcap = row[1]
        mcap_list.append(mcap)
    df2['date'] = time_list
    df2['mcap'] = mcap_list
        
    time_list=[]
    for row in info['total_volumes']:
        timeNum = row[0]
        timeNum = timeStamp(timeNum)
        time_list.append(timeNum)
        vol = row[1]
        vol_list.append(vol)
    df3['date'] = time_list
    df3['vol'] = vol_list
            
    df = pd.merge(df1,df2,on='date')
    df = pd.merge(df,df3,on='date')  
    df['crypto'] = crypto
    
    df_total = pd.concat([df_total, df], ignore_index=True)
    count += 1
    if count%50 ==0:
        time.sleep(60)

df_total.to_csv(path+os.sep+'df_total01.csv',encoding = 'utf_8_sig')
```

值得注意的是，免费版本API用户不能过于频繁的通过API访问数据，因此我特地加了一个计数器，每执行50次，就停顿1分钟时间。

至此，df_total就记录了所有加密货币从起始日至终止日的价格，市值以及交易量数据，总共包含4,329,182行数据，6列字段，存储结构如下：

![image](https://user-images.githubusercontent.com/93023212/170834235-a5d3c111-6acc-4be2-b28d-3c8cb38d4ebc.png)

如果要跑完整个代码，可能需要一整天的时间，因此我将所有加密货币分成三等分，通过与小组成员合作，分别用三部电脑获取数据，再合并到一起，记为all_data，这将节省大量时间。

# 数据清洗与处理

这部分工作可能是整个代码中最复杂最繁琐的部分，首先，我删除数据有缺失的行目，以减少后期无效数据处理量。

```python
all_data = all_data.sort_values(by='date')
all_data['date'] = pd.to_datetime(all_data['date'])

#Delete data without complete info
all_data = all_data.replace(0,np.nan).dropna()
```

接论文中的数据是以周为单位的，而我获取到的数据是以日为单位，这就意味着，我要将所有日数据改为周数据，并需要有字段区分不同的周。

这里我用了一个算法效率不高但是可行的办法，即设定一个开始日期，然后开始遍历所有的日期，若所遍历的日期属于开始日期的未来7天，则该日期记为第一周，接着开始日期加七天，重复上述流程，所遍历日期在新的时间段的，记为第二周，从而依次记录到最后一周，具体代码实现路径如下，预计用时15分钟：

```python
# Divide 7 date into a week
print('Dividing weeks...')
week_list=[]
for date in tqdm(all_data['date']):
    week=1
    begin_date = dt.strptime('20140101','%Y%m%d')
    end_date = begin_date + datetime.timedelta(days=7)
    while (date >= end_date) or (date<begin_date):
        begin_date = end_date
        end_date = begin_date + datetime.timedelta(days=7)    
        week+=1
    week_list.append(week)
all_data['week']=week_list
```

另外，还需要将价格信息转化为收益率，计算日收益率的方法为前后两天的价格差异除于前一天的价格，但现在所有信息都混在all_data中，如果直接通过滞后一期价格信息，则可能导致前后两天的价格为两种不同货币的价格，因此，我的思路是先挑选出所有加密货币，挑选出之后再做滞后和计算，最后将所有结果再合并到一起，这样就能解决上述问题，但由于首行数据没有上一期的数据，所以无法得出收益率，因此最后还需要将该部分缺失数据的行数删除。具体代码实现路径如下：

```python
crypto_list =list(all_data['crypto'].unique())
single_data = pd.DataFrame()
new_all_data = pd.DataFrame()

print('Computing daily return...')
for crypto in tqdm(crypto_list):
    single_data = all_data[all_data['crypto']==crypto]
    single_data['last_price'] = single_data['price'].shift(1)
    single_data['daily_ret'] = (single_data['price']-single_data['last_price'])/single_data['last_price']    
    new_all_data = pd.concat([single_data,new_all_data], ignore_index=True)

new_all_data = new_all_data.replace(0,np.nan).dropna()
```

在有了周，价格，市值，交易量和收益率这几个字段后，便可以对数据进行处理计算，得到我们所需要的因子和收益率。这部分的思路是，拎出单种加密货币，再从单种加密货币的dataframe中遍历拎出不同周的数据，依次进行计算，并添加至在一起。计算完周收益率以及四类主题因子后，再计算加密货币市场总市值以及市场风险溢价，用于后续的单因子模型中。这里用到的无风险利率为美国十年期国债利率，按照date与df_week合并在了一起。

计算加密货币周收益率的方法如下：

![image](https://user-images.githubusercontent.com/93023212/170850444-cda8b63e-543f-4ca8-9207-eb9d5b33b0c1.png)

因子的计算方法请参看前文中对因子的介绍，具体代码实现路径如下：

```python
#Compute the factor we need in later research
df_week = pd.DataFrame()
print('Computing Factors...')
for crypto in tqdm(crypto_list):
    crypto_data = new_all_data[new_all_data['crypto'] == crypto]
    AGE = len(crypto_data)
    
    for week in list(set(week_list))[1:]:
        data = crypto_data[crypto_data['week'] == week]        
        if data.empty:
            pass
        else:
            #计算收益率以及规模，交易量和波动性三类因子
            flag=1        
            for dret in data['daily_ret']:
                flag = flag * (dret + 1)
            flag = flag -1
            df_week = df_week.append([{'crypto':crypto,
                                       'date':data['date'].iloc[-1],
                            'week':week,
                            'PRC':np.log(data['price'].iloc[-1]+1),
                            'MAXDPRC':max(data['price']),
                            'MCAP':np.log(data['mcap'].iloc[-1]+1),
                            'mcap':data['mcap'].iloc[-1],
                            'AGE': AGE,
                            'VOL':np.log(np.average(data['vol']+1)),
                            'PRCVOL':np.log(np.average(data['vol']*data['price'])+1),
                            'STDPRCVOL':np.log(np.std(data['vol']*data['price'])+1),
                            'week_ret':flag}],ignore_index=False)


#计算动量因子
df_week_with_lag=pd.DataFrame()
print('Computing laging data....')
for crypto in tqdm(crypto_list):
    crypto_data = df_week[df_week['crypto'] == crypto]
    crypto_data['mcap_lag1'] = crypto_data['mcap'].shift(1)
    crypto_data['week_ret_lag1'] = crypto_data['week_ret'].shift(1)
    crypto_data['week_ret_lag2'] = crypto_data['week_ret'].shift(2)
    crypto_data['week_ret_lag3'] = crypto_data['week_ret'].shift(3)
    crypto_data['week_ret_lag4'] = crypto_data['week_ret'].shift(4)
    crypto_data['week_ret_lag1-4'] = (crypto_data['week_ret_lag1'] +1 )*(crypto_data['week_ret_lag2'] +1 )*(crypto_data['week_ret_lag3'] +1 )*(crypto_data['week_ret_lag4'] +1 ) -1
    df_week_with_lag = pd.concat([crypto_data,df_week_with_lag],ignore_index = 'True')
df_week = df_week_with_lag
df_week = df_week.dropna()

#计算该周整体加密货币市场的市值
week_vs_mcap = df_week.groupby('week')['mcap'].sum()
week_vs_mcap = week_vs_mcap.to_frame()
week_vs_mcap.rename(columns={'mcap':'all_crypto_mcap'},inplace =True)
week_vs_mcap['all_crypto_mcap_lag1']= week_vs_mcap['all_crypto_mcap'].shift(1)
week_vs_mcap['Rm'] = (week_vs_mcap['all_crypto_mcap'] - week_vs_mcap['all_crypto_mcap_lag1'] ) / week_vs_mcap['all_crypto_mcap_lag1']

df_week = pd.merge(week_vs_mcap,df_week,on = 'week')

#计算市场整体收益率Rm，载入合并Rf文件，并将年利率转化为周利率。
df_week['date'] = list(map(lambda x: x.replace(' 08:00:00',''),df_week['date']))
df_week['date'] = pd.to_datetime(df_week['date'],format = "%Y-%m-%d")
df_rf = pd.read_excel(path+os.sep+'US_T-bills.xlsx')
df_week = pd.merge(df_week,df_rf, on='date')
df_week['Rf'] = df_week['Rf']/100/52

df_rf = df_week.groupby('week')['Rf'].mean()
df_rf = df_rf.to_frame()
df_rf.rename(columns={'Rf':'week_Rf'},inplace = True)
df_week = pd.merge(df_week,df_rf, on='week')
df_week['CMKT'] = df_week['Rm'] - df_week['week_Rf']

df_week = df_week.dropna()
print(df_week)
df_week.to_csv(r'C:\Users\hp\Desktop\QT Final'+os.sep+'df_week.csv',encoding='utf_8_sig')

```

至此，获取数据及数据清洗处理工作全部完成，我们需要的数据字段都生成完毕了，现在的df_week仅剩309715行，拥有22个字段。

```
Input：df_week.columns
Output: 
Index(['Unnamed: 0', 'week', 'all_crypto_mcap', 'all_crypto_mcap_lag1', 'Rm',
       'crypto', 'date', 'PRC', 'MAXDPRC', 'MCAP', 'mcap', 'AGE', 'VOL',
       'PRCVOL', 'STDPRCVOL', 'week_ret', 'mcap_lag1', 'week_ret_lag1',
       'week_ret_lag2', 'week_ret_lag3', 'week_ret_lag4', 'week_ret_lag1-4'],
      dtype='object')
```

后续内容，敬请期待技术博客的第二部分。














