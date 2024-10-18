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

We can see that in practice using the function **correlation_matrix_data** from **utilities**: ![adjacent_lookbacks](https://github.com/user-attachments/assets/0742ea85-2fe4-4b6c-aa51-4039c25f0cd6)

**Note**: We see that the formula **1/sqrt(a)** doesn't work for pairs of adjacent lookbacks. This is because this formula works on detrended price series, but we didn't do that. We approximated that by normalizing for price and recent volatility. 
Nevertheless the symmetry for adjacent lookback periods still works.

The range of the lookback windows will be from 2 to 256. The reasoning is simple: we don't know which lookback periods are the best for predicting future returns so we will try to include as many periods as possible using the previous structure in order to have an idea of the diversification (correlations) between the rule variations (different lookback window pairs).

After that our job is almost done for this rule with two major notes.

First of all on a portfolio level we want to have a universal scale in order to size accordingly our position (this is again not my idea, just an implementation from **Robert Carver**'s book). This means that we want all our trading universe to have the same values when longing/shorting a coin. An easy way to implement this, is to scale the difference so that the mean of the absolute values of all the differences is 10. So +10 is an average buy and -10 an average sell. A forecast of +5 would be a weak buy, and -20 is a very strong sell. Later, depending on our forecast score and the expected volatility we will size our positions. Since we don't want extreme forecast values to gives us an enormous posiziton size, we will cap it at double the expected absolute value, which in our case is +/-20. 
The forecast scalars are calculated by taking many coins, constructing the EWMAC rule for all the lookback windows and then taking the **average for every (lookback) pair of the absolute value of the forecasts**. Then we divide 10 (which is the expected value that we want our signals to have) with that **average** for every pair. This is our **forecast scalar**. Then we go back to the signal construction and multiply our rule variation with that **scalar**. We achieve this by using the **calculate_vol_adjusted_ewmac** and **ewmac_scalar** functions. 

Below is the distribution of of our average forecast for different symbols and different ewmac variations: ![ewmac_vars_distr](https://github.com/user-attachments/assets/97e4f453-90a8-44af-bc61-ac6e05f7f45c)

![ewmac_vars_distr_2](https://github.com/user-attachments/assets/16c08dbd-5d4d-4970-8232-4b27f352cbbd)
**Note:** The very low mean signal to the left is due to (WIF/USDT) pair which was listed in February of 2024, so we haven't see te full distribution of the signals for this symbol.

We see that it works as expected and the scaling applies uniformly across our assets.

Lastly, since we don't know which one of these works better and we don't want to overfit by picking the best based on backtested performance, we will try something else. We will weight them according to their correlations, which we shown for EWMAC ruyle on the 
beginning. 
Since we don't want to use a computational expensive method like bootstrapping, we will weight them according to the **handcrafting method** from **Robert Carver**'s book, shown in this exempt from his book:
![handcrafting](https://github.com/user-attachments/assets/cd30cb67-3187-48ef-b88b-24838a688c07)

We will split the rule variations in two groups. The first will be the shorter lookback pairs: 2_8, 4_16, 8_32 and the second will longer ones: 16_64, 32_128, 64_256. This way we can apply the handcrafting method by assigning the weights 42%, 16% and 42% in that order for the corresponding element in the two groups. Then we will divide them by 2, since we have two groups of weights. So the final weights should be: 
![ewmac_starting_weights](https://github.com/user-attachments/assets/13fa5b83-a1a9-4ca4-bc9c-2bec4c784185)

At this stage, we have to remember that trading costs. For that reason we want to minimize our trading costs and consequently we need to know the turnover for every rule variation and include this in our weighting decision. We will use another function from **utilities**
which is called **rule_turnover**. We already suspect that the very fast lookback pairs (i.e. 2_8) will have bigger turnover than the slower pairs. In addition, these are information that could have been known ex ante, when we began to trade this strategy. 
With that in mind we will show the yearly turnover aggregated for all our trading universe at this date (16th of October, 2024) and at the first year of each symbol's listing. 
![dnld](https://github.com/user-attachments/assets/3c02c31b-4591-41fc-9952-c816ff849eeb)


It is clearly shown that the the fastest pair constantly switches forecast, which incurs costs due to position adjustments.

Also we can visualize for a single asset (BTC/USDT) the different forecast variations:
![side_2_side_turnover](https://github.com/user-attachments/assets/605a8de8-e2c6-4739-a1e7-44713cf9e863)


For that reason and since the two rules variations are highly correlated: ![corr](https://github.com/user-attachments/assets/eb2e7262-ac6c-4332-8e18-1517057b34de)


we will change the weights between 2_8 and 4_16 variation, in order to reduce trading costs. (We did the same for the 16_64 and 32_128 even though the turnover difference isn't that pronounced, for symmetry reasons. Either way this decision doesn't step from any
information that we couldn't have at the construction of our signal creation.)

Finally we can combine our EWMAC variation into a single one with the weights that we chose. The combined signal is given by the code below:

![ewmac_combined](https://github.com/user-attachments/assets/39a76370-ac39-408a-a1fb-7afb96f69cc4)

The scalar **1.30** we used at the second cell of code is to scale the combined signal so the average is still +10. To calculate this we use an excerpt, again from **Robert Carver**'s book, and apply the code below:

![comb_sig_theor_code](https://github.com/user-attachments/assets/3fcf6eb6-79e6-4e71-be64-40c6023294c7)

Below is the resulting distribution of the combined signal:

![combined_ewmac_distr](https://github.com/user-attachments/assets/2932bd43-58c4-43f2-a4f2-f617697a392b)


Using the same methodology we will create two more signals, **bolmom** and **breakout**. 
