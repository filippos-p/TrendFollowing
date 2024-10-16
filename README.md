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

The universe selection happens at this step. The choice was made based on the top 50 by Market Capitalization from **https://coinmarketcap.com**. The reason being will be explained later in the research process, but a brief explanation is that trend works better on lower volatiltiy coins, and the smaller the Capitalization the higher the volatility due to lack of liquidity in the orderbooks. Also we will use the USDT pairs since these are the most liquid trading pairs.  Another note is that we used less than 50 coins, since some of the top 50 were stablecoins and some others weren't available on Binance for trading. 
Stablecoins are cryptocurrnecy coins that are pegged to their respective currency. For example USDT is pagged to USD (United States Dollar).

Lastly we will save our data (which are a Dictionary of DataFrames) in a pickle file, for ease of access.



**signals:**

At this stage we want to create our signals or trading rules that hopefully can capture the momentum effect. What is momentum and why it happens is a good question, that has varying answers. We won't bother trying to explain possible reason for its existence. 
Our assumption is that momentum exists in markets and is especially pronounced in Crypto markets and we will try to create signals that can exploit this effect.

Our first rule is taken from **Robert Carver's book Systematic Trading: A unique new method for designing trading and investing systems** and is called **EWMAC**. A form of EWMAC is widely used by CTAs (trend following funds) and is basically a moving average crossover, in our case we take an exponential moving average. We calculate the exponential moving average of the closing prices on a rolling basis. The crossover is the difference between the fastest (smaller lookback window) minus the slowest (bigger lookback window) exponential
moving average (from now on we will use exponential moving average and moving average interchangeably). Note that we take the difference and not just a whether one is bigger than the other. This is because we will construct continuous signals and not binary. The reasoning is that a continuous signal provides more information about the strength of the signal than a mere 0 or 1. Our goal is to have different posiziton size depending on the signal strength, so stronger signals will have more bigger position sizes and weaker ones will have smaller size. Finally we will standarize the moving average, using the standard deviation of returns on a rolling basis. The standard deviation of returns is basically the volatility of our asset.

How are we choosing the lookback windows? First of all there is a heuristic (taken from **https://twitter.com/macrocephalopod**) that says that for two trend signals with lookbacks n1 and n2 on the same asset the correlation of the signals is approximately min(n1, n2)/sqrt(n1 * n2). This means that if we pick the lookbacks in geometric progression (i.e. 2, 4, 8, 16, 32, 64, ...) then any pair of adjacent lookbacks will have the same correlation (which will be 1/sqrt(a), where a is the ratio of the lookback pairs). 

We can see that in practice using the function **correlation_matrix_data** from **utilities**: ![adjacent_lookbacks](https://github.com/user-attachments/assets/c1a8d6b0-a710-490c-9b67-b5cab631c8b7)
**Note**: We see that the formula **1/sqrt(a)** doesn't work for pairs of adjacent lookbacks. This is because this formula works on detrended price series, but we didn't do that. We approximated that by normalizing for price and recent volatility. 
Nevertheless the symmetry for adjacent lookback periods still works.

The range of the lookback windows will be from 2 to 256. The reasoning is simple: we don't know which lookback periods are the best for predicting future returns so we will try to include as many periods as possible using the previous structure in order to have an idea of the diversification (correlations) between the rule variations (different lookback window pairs).

After that our job is almost done for this rule with two major notes.

First of all on a portfolio level we want to have a universal scale in order to size accordingly our position (this is again not my idea, just an implementation from **Robert Carver**'s book). This means that we want all our trading universe to have the same values when longing/shorting a coin. An easy way to implement this, is to scale the difference so that the mean of the absolute values of all the differences is 10. So +10 is an average buy and -10 an average sell. A forecast of +5 would be a weak buy, and -20 is a very strong sell. Later, depending on our forecast score and the expected volatility we will size our positions. Since we don't want extreme forecast values to gives us an enormous posiziton size, we will cap it at double the expected absolute value, which in our case is +/-20. 
The forecast scalars are calculated by taking many coins, constructing the EWMAC rule for all the lookback windows and then taking the **average for every (lookback) pair of the absolute value of the forecasts**. Then we divide 10 (which is the expected value that we want our signals to have) with that **average** for every pair. This is our **forecast scalar**. Then we go back to the signal construction and multiply our rule variation with that **scalar**. We achieve this by using the **calculate_vol_adjusted_ewmac** and **ewmac_scalar** functions. 

Below is the distribution of of our average forecast for different symbols and different ewmac variations: ![ewmac_vars_distr](https://github.com/user-attachments/assets/57dc68cb-8c9a-4aa9-b285-9b7e6c3dfa1b)

![ewmac_vars_distr_2](https://github.com/user-attachments/assets/8cfbf0dc-70ef-44a7-b684-b173d611c3b0)

We see that it works as expected and the scaling applies uniformly across our assets.

Lastly, since we don't know which one of these works better and we don't want to overfit by picking the best based on backtested performance, we will try something else. We will weight them according to their correlations. 
