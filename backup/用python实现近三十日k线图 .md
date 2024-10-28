```
import akshare as ak
import pandas as pd
import mplfinance as mpf

# 股票代码，例如“002236”代表某只股票
stock_code = "002236"

# 获取股票日 K 线数据
data = ak.stock_zh_a_hist(symbol=stock_code, period="daily", adjust="qfq")  # qfq 表示前复权数据

# 查看数据结构
print(data.head())
print(data.columns)

# 确保列名符合要求
data.rename(columns={"日期": "Date", "开盘": "Open", "收盘": "Close", "最高": "High", "最低": "Low", "成交量": "Volume"}, inplace=True)

# 将 "Date" 列转换为日期时间格式并设置为索引
data['Date'] = pd.to_datetime(data['Date'])
data.set_index('Date', inplace=True)

# 选择最近30天的数据
recent_data = data.tail(30)  # 选择最近30日的数据

# 绘制 K 线图
mpf.plot(recent_data, type='candle', volume=True, title=f"{stock_code} K线图", style='charles', mav=(3, 6, 9))
```

![Figure_1](https://github.com/user-attachments/assets/0be77dbf-e9cd-474b-8b6e-9a96bf6aeeb9)

