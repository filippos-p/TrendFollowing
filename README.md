In this repo, we will show some of the findings from momentum research on Cryptocurrency markets and then provide a simple idea on portfolio construction.
For this process four files will be created:

      1) data_fetching : will download and store the data that we need for our analysis.
      
      2) signals : will showcase the construction of our momentum signals.
      
      3) utilities : will assist with choosing appropriate signal weights in the portfolio.
      
       4) research : will show our findings.


**data_fetching:**

The first two functions (**generate_dates**, **split_list_into_chunks**) are created due to Binance's API limit requests. For the data that we need there is a limit of 1000 data points per request.
In our research we will use daily data from 2018, with each day corresponding to one data point. So we need to split our dates in batches of 1000 days.

Then we will use the function **fetch_klines** that performs one request (of 1000 data points) and pulls data for a single coin. 
From these data we will mostly use:
            **timestamp** which contains the open time in unix format (it is transformed in datetime format for readability)
            and
            **close** which is the last traded price (so happens on the closing time) for that day (identified by the opening time)
            
Here we need to note that the closing price is **not tradeable**, so any assumptions based on trading at that price are baseless. But we won't use it for that purposes.

The function **fetch_klines_for_symbols** performs the previous operation for multiple symbols simultaneously using a ThreadPoolExecutor and also applies the **split_list_into_chunks** function in order to retrieve data for longer than 1000 days. The universe selection happens at this step. The choice was made based on the top 50 by Market Capitalization from **https://coinmarketcap.com**. The reason being will be explained later in the research process, but a brief explanation is that trend works better on lower volatiltiy coins, and the smaller the Capitalization the higher the volatility due to lack of liquidity in the orderbooks. Also we will use the USDT pairs since these are the most liquid trading pairs.  Another note is that we used less than 50 coins, since some of the top 50 were stablecoins and some others weren't available on Binance for trading. 
Stablecoins are cryptocurrnecy coins that are pegged to their respective currency. For example USDT is pagged to USD (United States Dollar).

Lastly we will save our data (which are a Dictionary of DataFrames) in a pickle file, for ease of access.

