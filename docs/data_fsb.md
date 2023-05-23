---
title: Data
layout: home
parent: Docs
nav_order: 1
---

# pysystemtrade-fsb data

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

This fork includes three entirely new MongoDB collections, `epic_history`, `market_info`, and `epic_periods`. Epic history is an Arctic collection, the other two are normal Mongo collections. They all relate to [the challenges with the way IG offer their bets](https://github.com/bug-or-feature/pysystemtrade-fsb#challenges).

### epic_history

Epic history is a timeseries dataframe, one per instrument, and records the changes in the relationship over time between an instrument, its EPICs, and its underlying futures contracts. Here's a sample from `GOLD_fsb` at the time of writing (May 2023):

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

To understand what this means, we need to look at the full EPIC. For GOLD_fsb, all the known EPICs are currently

- MT.D.GC.Month1.IP
- MT.D.GC.Month2.IP
- MT.D.GC.Month3.IP

The EPIC can be split into 5 parts, separated by the dot '.' character. Part 2 is always 'D', and part 5 is always 'IP'. Part 3 describes the instrument, 'GC' here, which is the two letter symbol for the Gold future on the CME. Part 1 represents the asset class, Indexes, Rates, Commodities, Energy, Metals etc. Part 4 we call the 'epic period', and it maps to the column names in the 'epic_history' Arctic collection. 

Looking at the last row in the collection, the Month1 column shows us that the EPIC `MT.D.GC.Month1.IP` currently represents the June 2023 Gold contract, it expires on 25 June 2023, and it is currently tradeable. The previous row shows us that before 18 April 2023, the June contract's trading status was EDITS_ONLY. 

The Month3 column shows that the next contract, August 2023, will have the EPIC `MT.D.GC.Month3.IP`, but it is currently not tradeable. The Month2 column shows that the December 2023 contract will have the EPIC `MT.D.GC.Month2.IP`, (also not tradeable), but before the end of March, that EPIC represented the April 2023 contract. 

The first column shows us that before the middle of February 2023, there was another epic period being used. The name changed from 'MONTH1' to 'Month1' around that time. 

### market_info

Market info stores the entire response from IG's `/markets` REST endpoint. The response contains all the published information for the passed EPIC. For example, minimum bet size, market hours, expiry date, margin bands, dealing rules and so forth. We save it locally, so we don't have to hit the IG API too often, due to the rate limits. 

### epic_periods

TBD 


