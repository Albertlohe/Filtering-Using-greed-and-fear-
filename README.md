# Filtering-Using-greed-and-fear-

1.	random.seed(self.get_parameter('seed', 30))
2.	self._probability_of_trade = 0.01
3.	self._filter = bool(self.get_parameter('filter', 0))



1.	self._index = self.add_data(FearGreedIndex, 'FG')
2.	self._index.history = self.history(self._index.symbol, datetime(2014, 7, 1), self.time).loc[self._index.symbol].qcindex
3.	self._index.regime = None

