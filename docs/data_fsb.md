---
title: FSB data
layout: home
parent: Docs
---

# pysystemtrade-fsb data setup

In pysystemtrade-fsb the way data works is a little different from the upstream project. In this project we import and store individual Futures contract prices as normal. We then generate the individual Futures Spread Bet (FSB) contract prices from the futures prices; the futures prices are not overwritten. Multiple and adjusted prices are only generated for FSB instruments.

We do this for a few reasons:
- it is not practical to download historical FSB prices (at least from IG, the
 only currently supported broker)
- it would be easier to start trading Futures, at some point in the future, if 
 required  
- it would be easier to start trading some other futures-based derivative product 
 (CFDs?), at some point in the future, if required  

Roll calendars are different too. For many bets only the front contract is available, expiry dates are often different, and so on. We try to make the FSB roll calendars match the ideals of the futures ones, in terms of carry contract selection, how long before expiry to roll etc. But it's usually not completely possible, so we try to get as close as we can.

These docs assume you have some external script to download futures contract prices. I use [this](https://github.com/bug-or-feature/bc-utils). I also have scripts to download historic FSB prices from IG (at a low resolution), but this isn't strictly necessary; the FSB production scripts will do this once you have started sampling an instrument.


## setup process

TBD
