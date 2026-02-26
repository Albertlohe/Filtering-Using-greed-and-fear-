Introduction
Warren Buffett once said, “A simple rule dictates my buying: Be fearful when others are greedy, and be greedy when others are fearful”. In this research, we test Buffett’s advice by applying a Markov switching regression model to the new Fear and Greed Index dataset to detect if the current regime of the US Equity market is rife with fear. The results show that limiting long-biased trades to periods of time where the market is fearful tends to increase the net profit of those trades.

Background

A trade filter is a condition that dictates whether or not a strategy can open new positions. A popular example of this may be prohibiting trades during periods of high VIX, or only entering long positions with the S&P500 is above its 200-day moving average. While scanning for trades, a filter prohibits long positions during bearish regimes and prohibits short positions during bullish regimes to improve the probability of successful trades.
Introducing the Fear and Greed Index
To emulate Buffett’s process of being greedy when others are fearful, we can use the new Fear and Greed Index to construct a trade filter. This dataset is an index that represents the degree of fear and greed in the US Equity market. The index is composed of the following indicators:
1.	Market momentum: The difference between the S&P 500 price and its 125-day SMA.
2.	Stock price strength: The difference between the number of stocks trading at 52-week highs and the number of stocks trading at 52-week lows, divided by the number of stocks in the market.
3.	Stock price breadth: The McClellan Volume Summation Index of liquid stocks on the NYSE.
4.	Options put/call ratio: The 5-day SMA of the put/call ratio across all stocks in the market.
5.	Market volatility: The difference between the daily VIX value and its 50-day SMA.
6.	Safe haven demand: The difference between the trailing 20-day return of stocks (SPY) and the trailing 20-day return of bonds (IEF).
7.	Junk bond demand: The difference between the yield of junk and investment-grade bonds.
Each indicator is transformed to a value between 0 (fearful) and 100 (greedy) based on how much the current value deviates from its historical mean, normalized by the magnitude of historical deviations. The index then equal-weights the normalized value of each indicator to calculate the final value for the index. Our index is an almost perfect match to a popular closed-source index with the same name.
 
To classify the current regime without introducing a numerical parameter, we can use a Markov switching regression model (MSRM) to detect if the current environment is fearful or greedy.
Markov Switching Regression Models
An MSRM with no exogenous regressors models a time series as an intercept with some error, where the intercept is unique to each regime. It’s defined as
rt=μSt+εtεt∼N(0,σ2)
where St∈{0,1} and the regime transitions according to
P(St=st|St−1=st−1)=[p001−p00p101−p10]
To estimate the p00, p10, μ0, μ1, and σ2 parameters, the model uses maximum likelihood estimation.
Quantifying the Value of a Filter
To determine if a filter improves a strategy, we can compare the backtest result of the strategy with the filter and the backtest result of the strategy without the filter. The filter may improve the returns of a strategy, but degrade the returns of a different strategy. Therefore, it’s important to test the value of the filter across many strategies. To accomplish this without introduction bias, we can run the test with many strategies that place random trades.

Implementation

To implement this filter test, we start with the initialize method, where we set a random seed, define the probability of trading each minute, and enable or disable the filter.
1.	random.seed(self.get_parameter('seed', 30))
2.	self._probability_of_trade = 0.01
3.	self._filter = bool(self.get_parameter('filter', 0))
Next, we add the Fear and Greed Index, get its history, and create a member to track the current regime.
1.	self._index = self.add_data(FearGreedIndex, 'FG')
2.	self._index.history = self.history(self._index.symbol, datetime(2014, 7, 1), self.time).loc[self._index.symbol].qcindex
3.	self._index.regime = None
Then, we add the SPY to trade and remove fees.
1.	self._equity = self.add_equity('SPY')
2.	self._equity.set_fee_model(ConstantFeeModel(0))
To end the initialize method, we add a Scheduled Event to liquidate the portfolio at market close to avoid holding overnight.
1.	self.schedule.on(self.date_rules.every_day(self._equity.symbol), self.time_rules.before_market_close(self._equity.symbol, 1), self.liquidate)
 
The on_data method runs every minute the market is open and every day when the Fear and Greed Index updates. If the market is open, a trade is signalled, the filter is enabled, and the current regime is “fear”, then the algorithm places a trade. If the filter is disabled, the algorithm places the trade regardless of the current regime.
1.	if self._equity.symbol in data and self._equity.exchange.hours.is_open(self.time + timedelta(minutes=1), False):
2.	    trade = random.random() < self._probability_of_trade
3.	    if not trade:
4.	        return
5.	    if self._filter and self._index.regime == 1: 
6.	        return
7.	    self.market_order(self._equity.symbol, 100 if not self.portfolio.invested else -100)
Lastly, when the Fear and Greed Index updates, the algorithm updates the index history and fits the MSRM to detect the current regime.
1.	if self._index.symbol not in data:
2.	    return
3.	self._index.history.loc[self.time] = self._index.close
4.	regimes = pd.Series(
5.	    MarkovRegression(self._index.history, k_regimes=2).fit().smoothed_marginal_probabilities.values.argmax(axis=1), 
6.	    index=self._index.history.index
7.	)
8.	self._index.regime = regimes.iloc[-1]
+ Expand

Results

We ran an optimization job from July 2015 to April 2025 to ensure that we had at least 1 trailing year of the Fear and Greed Index values to fit the MSRM. We tested the algorithm with the filter enabled and disabled. We also tested random seeds from 30 to 50. The following image shows the heatmap of net profit for the parameter combinations:
 
The results show that enabling the filter increased the net profit for 18/21 (85.7%) random seeds. In conclusion, limiting long-biased trades to periods of time that an MSRM detects the Fear and Greed Index is low (a fearful market environment) usually increases the net profit of intraday strategies
