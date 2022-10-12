# Skip
Harvard Final Project Code | Forex USD Trading Bot
#region imports
from AlgorithmImports import *
#endregion
import numpy as np 

class USD (QCAlgorithm):

    def Initialize(self):
        # Set the cash for backtest
        self.SetCash(3000)
        
        # Start and end dates for backtest
        self.SetStartDate(2017,10,21)
        self.SetEndDate(2022,10,27)
        
        # Add asset
        self.symbol = self.AddEquity("USD/AUD", "USD/EUR", "USD/GBP", "USD/JPY", "USD/CAD", "USD/NZD", "USD/CHF", Resolution.Daily).Symbol
        
        # Lookback length for b/o (in days)
        self.lookback = 30
        
        # Upper/lower limit for lookback length
        self.ceiling, self.floor = 90, 60
        
        # Price offset for stop order
        self.initialStopRisk = 0.97
        self.trailingStopRisk = 0.90
        
        # Schedule function 20 minutes after Tokyo market open
        self.Schedule.On(self.DateRules.Weekdays(self.symbol), \
                        self.TimeRules.AfterTokyoMarketOpen(self.symbol, 20), \
                        Action(self.EveryMarketOpen))


    def OnData(self, data):
        # Plot security's price
        self.Plot("USD Chart", self.symbol, self.Securities[self.symbol].Close)

 
    def EveryMarketOpen(self):
        # Dynamically determine lookback length based on 30 day volatility change rate
        close = self.History(self.symbol, 31, Resolution.Daily)["close"]
        todayvol = np.std(close[7:11])
        yesterdayvol = np.std(close[7:95])
        deltavol = (todayvol - yesterdayvol) / todayvol
        self.lookback = round(self.lookback * (1 + deltavol))
        
        # Account for upper/lower limit of lockback length
        if self.lookback > self.ceiling:
            self.lookback = self.ceiling
        elif self.lookback < self.floor:
            self.lookback = self.floor
        
        # List of daily highs
        self.high = self.History(self.symbol, self.lookback, Resolution.Daily)["high"]
        
        # Buy in case of breakout
        if not self.Securities[self.symbol].Invested and \
                self.Securities[self.symbol].Close >= max(self.high[:-1]):
            self.SetHoldings(self.symbol, 1)
            self.breakoutlvl = max(self.high[:-1])
            self.highestPrice = self.breakoutlvl
        
        
        # Create trailing stop loss if invested 
        if self.Securities[self.symbol].Invested:
            
            # If no order exists, send stop-loss
            if not self.Transactions.GetOpenOrders(self.symbol):
                self.stopMarketTicket = self.StopMarketOrder(self.symbol, \
                                        -self.Portfolio[self.symbol].Quantity, \
                                        self.initialStopRisk * self.breakoutlvl)
            
            # Check if the asset's price is higher than highestPrice & trailing stop price not below initial stop price
            if self.Securities[self.symbol].Close > self.highestPrice and \
                    self.initialStopRisk * self.breakoutlvl < self.Securities[self.symbol].Close * self.trailingStopRisk:
                # Save the new high to highestPrice
                self.highestPrice = self.Securities[self.symbol].Close
                # Update the stop price
                updateFields = UpdateOrderFields()
                updateFields.StopPrice = self.Securities[self.symbol].Close * self.trailingStopRisk
                self.stopMarketTicket.Update(updateFields)
                
                # Print the new stop price with Debug()
                self.Debug(updateFields.StopPrice)
            
            # Plot trailing stop's price
            self.Plot("Data Chart", "Stop Price", self.stopMarketTicket.Get(OrderField.StopPrice))
