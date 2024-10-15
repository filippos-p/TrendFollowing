In this repo, we will show some of the findings from momentum research on Cryptocurrency markets and then provide a simple idea on portfolio construction.
For this process four files will be created:

       **data_fetching** : will download and store the data that we need for our analysis.
      
       **signals** : will showcase the construction of our momentum signals.
      
       **utilities** : will assist with choosing appropriate signal weights in the portfolio.
      
       **research** : will show our findings.


**data_fetching:**

The first two functions (generate_dates, fetch_listing_date) are created due to Binance's API limit requests. For the data that we need there is a limit of 1000 data points per request.
In our research we will use daily data from 2018, with each day corresponding to one data point. So we need to split our dates in batches of 1000 days.
