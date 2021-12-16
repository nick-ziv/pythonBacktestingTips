
# Improving backtesting through optimization and design choices

This post will be oriented toward the Python language

Part 1: Backtesting design:

One of the most important aspects of designing a backtesting system is the elimination of look-ahead bias.  This type of error is typically introduced through the design of a for-loop backtesting system where data that would not be available for making decisions is presented before it should have been, typically just one candle in advance.  To eliminate look-ahead bias in a for-loop backtest, you can handle order execution in the next iteration of the for-loop instead of in the current iteration:

    # create an order list to store order information between iterations

    order_list = []

    for current_candle in stock_data:

	    # handle order placement (for all stocks) from order_list using prices of current_candle
	
	    # handle creating new orders from signals (for all stocks) using data up to current_candle
	
	    # perform logging, etc.

In this style of loop, you should never use data that is further in the future than 'current_candle'.  As accuracy (not speed) is what a backtest is all about, correct results should be the top priority.  

## Danger of look-ahead bias:
A program that generates signals with look-ahead bias, when implemented in live trading, will generate signals that sometimes may not show up in the backtest.  This discrepancy needs to be avoided for obvious reasons.

## Candle Catalog:

Efficient creation of 'stock_data' list for backtest iteration:
A backtest needs to account for all trades made on all trading assets in chronological order to properly simulate your allocation rules.  An efficient way to complete this task involves what I call the 'candle catalog' approach, where the main backtest loop iterates over each entry in the candle catalog from start to finish.

**Important**: using the *.index* method or *'in'* keyword to search for items in lists should be avoided as they are very slow.  I recommend converting lists to dicts where the dict key is what you were searching for and the dict value contains the rest of the data.

## Candle Catalog Approach

    #raw_stock_data is in format: {'ticker': [[timestamp, o, h, l, c, v, ...], ...], ...}

    # create a set which will contain the timestamps of every trading
    uniqueTimestamps = set()

    for _, ticker in enumerate(raw_stock_data):
        # loop through every ticker in raw_stock_data

    for i in range(len(signals[ticker])):
        # loop though every candle in this ticker
        # add this timestamp to the python set (sets do not contain duplicates, so a union of all timestamps will be created)
        uniqueTimestamps.add(usedStockData[ticker].data[i][0])  

    # create dict from the set containing all uniqueTimestamps
    allTimesDict = {x: {'datetime': x, 'tickers': {}} for x in uniqueTimestamps}

    # add stock data tickers and index in data to the allTimesDict dict

    for _, sd_ticker in enumerate(raw_stock_data):
	# loop through every ticker in raw_stock_data
	
    for dIndex, usdRow in enumerate(raw_stock_data[sd_ticker]):
    	# loop through every candle in this ticker
    	
    	# add the index of the data to the allTimesDict
        allTimesDict[usdRow[0]]['tickers'][sd_ticker] = dIndex

        # usdRow[0] is the timestamp

    # sort the dict based on datetime (convert timestamps to unix so that they can be sorted numericallly.)
    periodCatalog = dict(sorted(allTimesDict.items(), key=lambda b: dateStringToUnixTimestamp(b[0])))
        
This process creates the **periodCatalog** which can be iterated through from beginning to end in chronological order while having access to the index of where stock data is (in the orig. raw_stock_data dict) in relation to the current timestamp.  Because it is a catalog, only the index of the stock data's position in the list is stored, but this code could easily be changed to store the data as well.  The advantage of this method over storing the stock data in the catalog is that this method gives you an index that can be subtracted from to retrieve past data where storing data in the catalog would make it hard to get the past data.

Implementing the candle catalog, your backtest loop should look like this:

    for _, tPeriodKey in enumerate(periodCatalog):
        tPeriod = periodCatalog[tPeriodKey]
    
        # data available in tPeriod: {
        #     "datetime": string,
        #     "tickers": {
        #          'example': int( index of current timestamp in raw_stock_data ),
        #          ...
        #     }
        # }
    
        # handle order placement
    
        # handle creating orders, etc (look at top of post for proper structure)


That basically covers the speed improvement for backtesting.  It is also possible to generate signals before the backtest and just use the signals in the backtest to achieve results.  This would allow the same backtest code to be re-used for multiple strategies.


# Part 2: Stock Data Server:

A very slow part of strategy creation can be loading and managing stock data.  I recommend using a local server that hosts your stock data as an API that is accessed by your backtesting or signal generating program.  The basic function of this type of program is to deliver the stock data when needed with speed.  

Under the hood, a simple setup has a database (sqlite) to organize stock data and keep track of when it was last updated.  The server then recieves requests for data and then checks to see if the data is up to date.  If it is not, the stock data server requests the updated data from your typical API (AplhaVantage, IEX, etc.) and then saves that updated data to the database and returns it to the requesting program.  The next time the data is requested (next time your run the backtests, etc) the data is loaded locally (from disc) instead of from the internet.  To optimize this to be faster, a cache of recently used stock data can be created to speed up the rate at which data is returned.

If you guys want, I have a basic version of this type of program I can upload to GitHub.


# Part 3: I am Speed

To make your experiments faster, you should rely on improving the flow of data and reduce the number of slow operations used.  Here are some ways to make your program faster:

 

1: Avoid using .append when dealing with lists.  If you know the length of your data, use this method:

    # create a list with the desired length
    my_list = [0] * len(dataList)

    for i in range(len(dataList)):
    	# perform whatever calculation
    	
        my_list[i] = calculation_result
    
This can be used for a variety of things such as creating indicators, etc.  Search your code for .append and then see how many you can replace with a pre-allocated list

2: Use Python's sets for eliminating duplicate entries of an element.  This was used in my periodCatalog example for getting a list of every timestamp without duplicates.  Convert back to a list if you need to.

3: Use the multiprocessing module to handle multiple simultaneous backtests.  For this, I would recommend having a database or similar structure for logging parameters used and results, as writing a file for every backtest result can add up quickly.  Also logging as you go prevents your results from being lost due to an error on the last backtest ;)  Remember: creating processes takes time and can actually slow down your program depending on how fast your function is.  Noobs: Read about the difference between Python threading and multiprocessing.

4: Become proficient with JSON and CSV reading and writing.  This is important in general because you can use files to dynamically create configurations to test and have your backtest run each configuration.  Remember: speed of the program is one aspect, but total time you spend to get results is another aspect.


# Part 4: Costs

I strongly recommend not spending money on cloud computing unless you really need to (Neural networks).  If your backtests are taking too long, spend the effort on improving your system instead of spending your money.  Your machine is more capable than you think.  From personal experience, the biggest limitation can be your computer's RAM for storing the stock data.  Use task manager, htop, etc to view how your program uses your system's resources.  It may be worth the $100+ for better RAM.  You can always sell used hardware but money spent on the cloud is gone forever.


I have run backtests on my Chromebook and can achieve results in a few seconds for simple strategies by taking advantage of these tips. (Backtesting strategy using daily candles dating back to 1980 for 190+- stocks)

Nicholas Zivkovic (Nick_ziv) December 16, 2021
