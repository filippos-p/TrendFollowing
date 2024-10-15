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

The function **fetch_klines_for_symbols** performs the previous operation for multiple symbols simultaneously using a ThreadPoolExecutor and also applies the **split_list_into_chunks** function in order to retrieve data for longer than 1000 days.

Lastly we will save our data (which are a Dictionary of DataFrames) in a pickle file, for ease of access.

