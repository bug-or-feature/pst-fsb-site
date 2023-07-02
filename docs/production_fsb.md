---
title: Production
layout: home
parent: Docs
nav_order: 3
---

# pysystemtrade-fsb production


## setup process

This is my cheat sheet of command and scripts for setting up data in production

```
# Initialise the spot FX data in MongoDB from .csv files
python sysinit/futures/repocsv_spotfx_prices.py

# Update the FX price data
>>> from sysproduction.update_fx_prices import update_fx_prices
>>> update_fx_prices()

# Check that you have got spot FX data present (option 3, option 33)
$ ./sysproduction/linux/scripts/interactive_diagnostics

# import market info
$ python sysinit/futures_spreadbet/fsb_market_info.py

# import epics history
$ python sysinit/futures_spreadbet/fsb_epics_history.py

# view expected roll calendar 
$ python sysinit/futures_spreadbet/fsb_rollcalendars_to_csv.py expect

# create roll calendar 
$ python sysinit/futures_spreadbet/fsb_rollcalendars_to_csv.py build

# import futures contract price data
$ python sysinit/futures_spreadbet/fsb_contract_prices.py

# generate fsb contract price data - copies futures data, multiplying/inverting where required
$ python sysinit/futures_spreadbet/fsb_from_futures_contract_prices.py

# create multiple prices
$ python sysinit/futures_spreadbet/fsb_multipleprices.py

# create back adjusted prices
$ python sysinit/futures_spreadbet/adjustedprices_from_mongo_multiple_to_mongo.py

# import spread cost data
$ python sysinit/futures_spreadbet/repocsv_spread_costs.py

# import historic IG FSB prices
$ python sysinit/futures_spreadbet/ig_fsb_contract_prices.py

# update fsb market info
>>> from sysproduction.update_fsb_market_info import update_fsb_market_info
>>> update_fsb_market_info()
>>> update_fsb_market_info(["EURIBOR-ICE_fsb"])

# update fsb epics
>>> from sysproduction.update_epics import update_epics
>>> update_epics()
>>> update_epics(["EURIBOR-ICE_fsb"])

# update sampled contracts
>>> from sysproduction.update_sampled_contracts import update_sampled_contracts
>>> update_sampled_contracts()
>>> update_sampled_contracts(['EURIBOR-ICE_fsb'])

# update historic prices
>>> from sysproduction.update_historical_prices import update_historical_prices
>>> update_historical_prices()
>>> update_historical_prices(['NASDAQ'])
>>> update_historical_prices(['BOBL', 'BUND'])

# generate FSB updates from Futures prices
>>> from sysproduction.generate_fsb_updates import generate_fsb_updates
>>> generate_fsb_updates()

# update multiple adjusted prices
>>> from sysproduction.update_multiple_adjusted_prices import update_multiple_adjusted_prices
>>> update_multiple_adjusted_prices()
>>> update_multiple_adjusted_prices(['GOLD_fsb'])
>>> update_multiple_adjusted_prices(["VIX_fsb", "DOW_fsb"])

# update IG FSB prices
>>> from sysproduction.update_historical_fsb_prices import update_historical_fsb_prices, update_historical_fsb_prices_single
>>> update_historical_fsb_prices()
>>> update_historical_fsb_prices_single(['NASDAQ_fsb'])
>>> update_historical_fsb_prices_single(['BOBL_fsb', 'BUND_fsb'])

# clone fsb data
from sysinit.futures_spreadbet.clone_fsb_data_for_instrument import *
clone_data_for_instrument(instrument_from='EURIBOR', instrument_to= 'EURIBOR-ICE', ignore_duplication=True)
```

## Interactive scripts

We have some [Click](https://click.palletsprojects.com/en/8.1.x/) wrappers around the interactive scripts. They are configured in the `entry_points` block at the end of `setup.py`. There is a [section in the upstream docs](https://github.com/robcarver17/pysystemtrade/blob/master/docs/production.md#scripts-under-other-non-linux-operating-systems) about this setup. To see the options:

```
$ pst
Usage: pst [OPTIONS] COMMAND [ARGS]...

Options:
  --help  Show this message and exit.

Commands:
  c  Interactive controls
  d  Interactive diagnostics
  f  Interactive FSB commands
  h  Interactive update historical prices
  p  Interactive update capital
  r  Interactive update roll status
  s  Interactive order stack
```

And to run Interactive Diagnostics:

```
$ pst d

 INTERACTIVE DIAGNOSTICS


0: backtest objects
1: View instrument configuration
2: logs, emails, and errors
3: View prices
4: View capital
5: View positions & orders
6: Reports

Your choice? <RETURN for EXIT> 
```

### Differences with upstream

#### Interactive diagnostics
- *View FSB epic history* under *View instrument configuration*
- *Individual FSB contract prices* under *View prices*
- *FSB report* under *Reports*

#### Interactive controls
- *Delete instrument from price tables* also deletes FSB related data 

#### Interactive update historical prices
- this script remembers which instruments you update. The when you're finished it also generates FSB contract price updates from the futures prices. It then also rebuilds the multiple and adjusted prices for each instrument that has changed

#### Interactive update roll status

Market data is updated for each epic before the roll report is run 

The roll report shows some additional info:
- the expiry dates for priced and forward contracts
- the IG trading status (TRADEABLE, EDITS_ONLY, OFFLINE etc) for the priced and forward contracts
- the number of epics for that instrument

On *Roll_Adjusted*, the script checks that the new forward contract has an epic defined. If it does not, the user is advised to back out of the roll process.
