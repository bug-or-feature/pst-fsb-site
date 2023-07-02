---
title: Data
layout: home
parent: Docs
nav_order: 1
---

# pysystemtrade-fsb data


## Challenges

### Anatomy of an epic

On the IG platforms, a market that you can trade on is identified by an *epic*. An epic is a unique text string containing letters, dots and numbers. A lot of the challenges faced in this project relate to these epics, so its worth looking at their anatomy. Here's an example, it is one of the epics for a futures based dated spread bet on the FTSE 100 equities index: `IX.D.FTSE.MONTH1.IP`

There's five parts to the epic 
1. Two letter sector indicator. `IX` for indices, `CF` for FX, `IR` for bonds and rates, `EN` for energy, `CO` for commodities, `MT` for metals, `IN` for volatility indices
2. Always `D`
3. Market name. Examples `FTSE` for FTSE 100, `C` for Corn, `GC` for Gold
4. Epic period. Two or more period indicators. Examples `Month1`, `MONTH2`, `MAR`, `DEC` 
5. Always `IP`

### epic recycling
`IX.D.FTSE.DAILY.IP` is the epic for the daily funded bet (DFB) on the FTSE 100 Index. Epics work fine for undated instruments like this, but for dated products they are less useful. For example, at the time of writing (January 2022), the epic for the spread bet based on the ASX Australia 200 Index future (March 2022 expiry) is `IX.D.ASX.MONTH3.IP`. And the contract expiring June 2022 is `IX.D.ASX.MONTH2.IP`, and September 2022 is `IX.D.ASX.MONTH4.IP`. No doubt December 2022 will be `IX.D.ASX.MONTH3.IP` again. Epics are recycled.

Many markets only have two epics. For example the NASDAQ future has:
- `IX.D.NASDAQ.MONTH3.IP`
- `IX.D.NASDAQ.MONTH4.IP`

even though there are four contracts per year. And the NIKKEI:
- `IX.D.NIKFUT.FAR.IP`
- `IX.D.NIKFUT.FAR3.IP`

**Problem one:** there is no pattern or system for the epic names. When this was queried with IG Support, their 
response was:

> The dealing desk choose which epic to use and when, unfortunately we are unable to confirm beforehand, this is something they do as and when needed. They are also able to make changes to which epic is used at any given moment, so I do advise to check the epic if you ever receive an error

**Problem two:** obviously, due to the epics being recycled, it is impossible to get historical data for futures based spread bets. At the time of writing (January 2022) the oldest ASX contract for which you can get price history is December 2021. 

**Problem three:** just in terms of data, there are two API endpoints that an automated trading system would be interested in; one to get current prices, and another to get historical prices. To make the problems described above even worse, with IG these two APIs are not synced. An epic used in the current price API will give you data for a different contract than the one for the historical price API.

Another request was made to IG Support for a schedule of the roll cycles for all the futures based spread bets. The response was:

> Your request of "Roll cycle for all Commodities, Indices, and Bonds and Rates" is not something we can provide. We will only be providing information based on the API reference. Some products may have the same epic, but there is no guarantee it will be the same always and its up to our dealing desk to manage that. And in such cases, we will not be informing clients of the changes so thats something you will need to take note of.

### IG API rate limits

The IG APIs have rate limits; you can only make a certain number of requests during a certain time period (eg minute, hour, week etc) depending on the request type. The limits for the LIVE environment are published [here](https://labs.ig.com/faq), but the limits for DEMO are lower, and have been known to change randomly and without notice. The limits, as well as the epic recycling described above, mean that it is not practical to get useful historical prices from the IG APIs

### pysystemtrade is a futures trading system

From one of [Rob's comments](https://github.com/robcarver17/pysystemtrade/issues/391#issuecomment-911441646) in an issue in the main project:

> Although I sort of originally envisaged pysystemtrade being multi asset, in practice the use of futures is now completely baked in. However it should work pretty well for anything that looks like a future: a dated spread bet being a good example of that

What this means for this fork is that backtesting works pretty much straight out of the box, as long as the instrument config, roll config and price data is good. Anything else, especially production stuff, will likely need work.

## prices

In pysystemtrade-fsb the way data works is a little different from the upstream project. IG offers spread bets at prices that are often different from the underlying future. For example, the price for *US Crude*, (CRUDE_W_fsb), is 100 times the price of the future. RICE_fsb is 1000 times the price of its underlying instrument. And, to get the price for CHF_fsb, we have to invert it (divide by 1), **and** multiply by 10,000. Look at the [Instrument List Report](../reports/instruments.html), to see which instruments have multipliers, and which are inverted. So, in this project we import and store individual Futures contract prices, and then use them to generate the individual FSB prices; the futures prices are not overwritten. Multiple and adjusted prices are only generated for FSB instruments.

We do this for a few reasons:
- it is not practical to download historical FSB prices (at least from IG, the only currently supported broker) due to the IG API rate limits
- it would be easier to start trading Futures, at some point in the future, if required
- it would be easier to start trading some other futures-based derivative product (CFDs?), at some point in the future, if required

We do also download actual historic prices from IG. We do that at a low resolution, however, due to the IG API rate limits. They are not used to build multiple or adjusted prices though; they're just used as a kind of safety check, to make sure that the prices we generate from futures remain aligned with the prices offered by the broker. We run some correlation tests in production so if there are discrepancies, we know about them quickly.

## data sources

By default, this project gets its futures prices from a different source: Barchart. The Interactive Brokers (IB) API is a pain to set up, configure, and keep running. There is no need to run it unless actually trading through IB. The Barchart website uses an internal web API to provide data for charts. This project uses that API to get daily and hourly futures prices. That Barchart API restricts users to 60 requests per minute, but it is free to use. Another wrinkle is that the upstream project has a design limitation - it assumes that the price data source **is** the broker. As a result, some modules, classes and functions in this project have unexpected or inconsistent names, for example
- `sysbrokers/IG/ig_futures_contract_price_data.py`

The source for FX prices is different too. We get FX prices from [Alphavantage](https://www.alphavantage.co/) - again for free. It doesn't really matter, as all the IG spread bets are denominated in GBP, so none of the FX stuff is used. We get daily FX price updates nevertheless, in case they're needed in the future. 

## rolling

Roll calendars are different too. For many bets only the front contract is available, expiry dates are often different, and sometimes the next contract is not tradeable until the previous one expires. IG offer the option of auto rolling dated bets, but only as an *all or nothing* option - it cannot be set per instrument. We try to make the FSB roll calendars match the ideals of the futures ones, in terms of carry contract selection, how long before expiry to roll etc. But it's usually not completely possible, so we try to get as close as we can.

## new mongo collections

This fork includes four entirely new MongoDB collections, `fsb_contract_prices`, `epic_history`, `market_info`, and `epic_periods`. FSB prices and epic history are Arctic collections, the other two are normal Mongo collections. They all relate to [the challenges with the way IG offer their bets](https://github.com/bug-or-feature/pysystemtrade-fsb#challenges).

### epic_history

Epic history is a timeseries dataframe, one per instrument, and records the changes in the relationship over time between an instrument, its epics, and its underlying futures contracts. Here's a sample from `GOLD_fsb` at the time of writing (May 2023):

|         Date         |                      MONTH1                       |                        Month2                        |                       Month3                       | Month1 |
|:--------------------:|:-------------------------------------------------:|:----------------------------------------------------:|:--------------------------------------------------:|:------:|
| 2023-02-09 19:13:05  | JUN-23  &#x7c; 2023-05-25 17:30:00 &#x7c; OFFLINE |APR-23 &#x7c;2023-03-28 17:30:00&#x7c;TRADEABLE|AUG-23 &#x7c;2023-07-26 17:30:00&#x7c;OFFLINE| JUN-23 &#x7c;2023-05-25 17:30:00&#x7c;OFFLINE|
|2023-02-17 11:16:55|UNMAPPED|APR-23&#x7c;2023-03-28 17:30:00&#x7c;TRADEABLE|AUG-23&#x7c;2023-07-26 17:30:00&#x7c;OFFLINE|JUN-23&#x7c;2023-05-25 17:30:00&#x7c;OFFLINE|
|2023-03-24 11:17:32|UNMAPPED|APR-23&#x7c;2023-03-28 17:30:00&#x7c;TRADEABLE|AUG-23&#x7c;2023-07-26 17:30:00&#x7c;OFFLINE|JUN-23&#x7c;2023-05-25 17:30:00&#x7c;TRADEABLE|
|2023-03-30 11:17:13|UNMAPPED|DEC-23&#x7c;2023-11-27 18:30:00&#x7c;OFFLINE|AUG-23&#x7c;2023-07-26 17:30:00&#x7c;OFFLINE|JUN-23&#x7c;2023-05-25 17:30:00&#x7c;TRADEABLE|
|2023-04-10 11:16:56|UNMAPPED|DEC-23&#x7c;2023-11-27 18:30:00&#x7c;OFFLINE|AUG-23&#x7c;2023-07-26 17:30:00&#x7c;OFFLINE|JUN-23&#x7c;2023-05-25 17:30:00&#x7c;EDITS_ONLY|
|2023-04-11 11:17:26|UNMAPPED|DEC-23&#x7c;2023-11-27 18:30:00&#x7c;OFFLINE|AUG-23&#x7c;2023-07-26 17:30:00&#x7c;OFFLINE|JUN-23&#x7c;2023-05-25 17:30:00&#x7c;TRADEABLE|
|2023-04-17 11:17:13|UNMAPPED|DEC-23&#x7c;2023-11-27 18:30:00&#x7c;OFFLINE|AUG-23&#x7c;2023-07-26 17:30:00&#x7c;OFFLINE|JUN-23&#x7c;2023-05-25 17:30:00&#x7c;EDITS_ONLY|
|2023-04-18 11:16:55|UNMAPPED|DEC-23&#x7c;2023-11-27 18:30:00&#x7c;OFFLINE|AUG-23&#x7c;2023-07-26 17:30:00&#x7c;OFFLINE|JUN-23&#x7c;2023-05-25 17:30:00&#x7c;TRADEABLE|

To understand what this means, we need to look at the full epic. For GOLD_fsb, all the known epics are currently

- MT.D.GC.Month1.IP
- MT.D.GC.Month2.IP
- MT.D.GC.Month3.IP

Looking at the last row in the collection, the Month1 column shows us that the epic `MT.D.GC.Month1.IP` currently represents the June 2023 Gold contract, it expires on 25 June 2023, and it is currently tradeable. The previous row shows us that before 18 April 2023, the June contract's trading status was EDITS_ONLY. 

The Month3 column shows that the next contract, August 2023, will have the epic `MT.D.GC.Month3.IP`, but it is currently not tradeable. The Month2 column shows that the December 2023 contract will have the epic `MT.D.GC.Month2.IP`, (also not tradeable), but before the end of March, that epic represented the April 2023 contract. 

The first column shows us that before the middle of February 2023, there was another epic period being used. The name changed from 'MONTH1' to 'Month1' around that time. 

### market_info

Market info stores the entire response from IG's `/markets` REST endpoint. The response contains all the published information for the passed epic. For example, minimum bet size, market hours, expiry date, margin bands, dealing rules and so forth. We save it locally, so we don't have to hit the IG API too often, due to the rate limits. 

### epic_periods

TBD 


