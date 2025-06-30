# DataEngineeringChallenge @ CFP FlexPower

## Background

At FlexPower, engineers:

- design, build and run reliable serverless systems.
- collaborate closely with traders and operations to build internal tools that help the company get the best outcomes
  on the markets, thus making renewables more profitable.
- work with market and asset data to generate insights and data-driven trading decisions.

This challenge is meant to give you a taste of the type of problems our engineering team has to solve with regard to
data.
It should help you decide if you might have fun working with us.

It is also an opportunity to demonstrate your technical skills and the ability to understand and work with our domain.

## Intro

One of FlexPower's main products is the so-called **RouteToMarket**: bringing the energy produced by renewables'
assets to the European electricity markets.
FlexPower signs contracts with renewable assets' owners, and then sells the energy produced by
these assets on the electricity market, hopefully profitably thanks to its trading expertise.

At the end of each month, FlexPower invoices the asset owners based on the production of their assets:
FlexPower pays the asset owner an agreed-upon price for each produced mwh, and the asset owner pays FlexPower
a fee for the services provided, per mwh.
There are different pricing and fee models, i.e., ways to compute the price and fee, to accommodate different
types of assets and customers.

FlexPower also has to monitor the performance of its trading activities while the trading is happening, so that
the trading desk can make informed data-driven decisions.

In this challenge, we will look at different aspects of marketing renewables and present different data
entities involved.

The goal will be to compute different financial flows and present them in a way that helps stakeholders
understand the performance of the portfolio and of single assets.

PS: FlexPower also provides supply customers with energy (so assets consuming energy, for example, EV charging
stations).
The structure is essentially the same, but for simplicity's sake we will consider only production in this challenge.

## Assets "base" data

With base data we usually mean static attributes of assets. These can be **technical** or **contractual**.
We call all assets traded by FlexPower the **portfolio**.

### Technical data

Anything that describes the technical aspects of the asset, for example, total capacity, exact location (coordinates)
and some properties that are specific to the asset type, for example, solar or wind. This data is usually used to
compute production forecasts for these assets.\
For this challenge, we will restrict ourselves to wind and solar assets, but in general there are more
types (biogas, hydro, batteries...) with their own specific technical data.

The technical data can be downloaded from the vpp (virtual power plant). The file name contains the timestamp
at which the file was created. A new file is produced daily between 12:30 and 13:00.

### Contractual data

Each asset is assigned to a contract with a customer.
Contractual data determines how assets are to be invoiced and is kept in the CRM.

The data is a csv and contains the following attributes:

- contract_id: a unique alphanumerical identifier for the contract.
- asset_id: a unique alphanumerical identifier for the asset.
- metering_point_id: a unique alphanumerical identifier for the asset's metering point.
- capacity: the installed capacity of the asset, in MW.
- technology: the technology used by the asset, can be either "solar" or "wind".
- contract_begin: the date when the asset's contract with FlexPower began.
- contract_end: the date when the asset's contract with FlexPower ends, can be none.
- price_model: the pricing model used by the asset, can be either "fixed" or "market".
- price: the price paid by FlexPower to the asset owner, in €/MWh, if asset in the "fixed" price model.
- fee_model: the fee model used by the asset, can be either "fixed_as_produced", "fixed_for_capacity" or "
  percent_of_market".
- fee: the fee paid by the asset owner to FlexPower, in €/MWh, if asset in the "fixed" fee model.
- fee_percent: percentage of the market value paid by the asset owner to FlexPower per mwh, in %, if asset in the
  "percent_of_market" fee model.

A new file is created every week, on Sunday, between 04:30 and 05:00.
The portfolio operations team might create new files on demand if some urgent changes need to be ingested.

## Infeed(s)

Given how the RouteToMarket product functions, infeed data plays a central role in our system.
There are different types of infeed data, with different functions and degrees of trustworthiness.

### Forecasts:

For each asset, FlexPower forecasts the electricity production for 15-minute delivery intervals,
representing the average power expected to be produced by the asset over a quarter-hour, in kilowatt.\
This forecast is updated every 15 minutes and allows FlexPower to know how much electricity will be available to sell
on the markets for a quarter-hour in the future. The latest forecast for each quarter-hour is calculated at the start
of each 15-minute delivery range, but the latest "tradeable" forecast for each asset is produced 15 minutes before
delivery start (due to trading gate closures).\
For example, the latest forecast for the delivery start `2025-06-08T12:00:00Z` has the version `2025-06-08T12:00:00Z`
but the latest tradeable forecast has the version `2025-06-08T11:45:00Z`.\
To query the forecasts, use the following code snippet:

```python
from src.vpp.client import get_forecast
import pendulum

f = get_forecast(
    asset_id="WND-DE-003",
    version=pendulum.datetime(2025, 6, 8, 8, 15, tz="Europe/Berlin"),
)
```

### Live measured infeed:

In most cases, the vpp provides a live measurement of the energy produced by single assets.
These measurements are available almost instantly, and the production, measured in kilowatt, has an irregular
resolution.

### Final-measured infeed

Each asset is connected to the electricity grid through a so-called "metering point".
The production measured at this point is communicated to FlexPower by the Distribution System Operator (DSO).
The data is available, in most cases, every day with values for the previous day.
The values here are used to invoice the asset owner, and we get data for all assets at once, in a csv file.
The unit is kilowatt.

### Redispatch

If too much energy is being produced by renewables within a certain area, the DSO might decide to turn off some assets.
Some of these assets might happen to be in our portfolio.

In that case we would receive a file containing a boolean timeseries that indicates if an asset was turned off.
If no file was received, we assume that the asset was running normally.

### Best-of-infeed

As we have shown above, there are multiple sources of infeed data. We want to define a quantity that represents
our best guess, at any moment in time, for the production of an asset.
The formula is as follows:

- the best source is the final measured production, as it is the legally binding volume used for invoicing.
- the second-best source is live-measured infeed, if available
- if none of the above is available, we can take the latest forecast to date. Even better would be including the
  redispatch flag so that production for quarter hours when redispatch was called is set to 0.

## Trading

### Energy Trading in a Nutshell

Energy trading happens in an exchange, a market where traders working for energy producers (solar plants,
nuclear power plants, ...) and consumers (B2C energy providers, big energy-consuming industries
like steel and trains...) submit orders to buy and sell energy.
One of the major exchanges in Europe is called [EPEX](https://en.wikipedia.org/wiki/European_Power_Exchange).

These orders consist of a volume (in Megawatt, rounded to one decimal) over a predefined period of time,
called delivery period (for example between 12:00, the delivery start and 13:00, the delivery end, on a given day)
and for a given price per megawatt hour (referred to as mwh).
If the prices of two orders with opposite sides match, i.e., the buy price is higher than the sell price, then, a trade
is generated.
For example, if the orderbook contains an order to sell 10 mw for 10 euros/mwh and another trader submits
an order to buy 5 mw for 11 euros/mwh, the orders are matched by the exchange and a trade is generated for 5 mw at
10 euros/mwh.

### Trades (or private trades)

A list of trades that our trading floor executed is provided by the exchange as a JSON.
Volumes are in megawatt, and prices are in euro per mwh.\
A trade also has a delivery start and a delivery end, defining the time range for which the volume was traded.
This range can either span an hour or a quarter-hour, but an hourly trade could also be seen as four quarter-hourly
trades. For example,

```
side=SELL volume=10.0 MW price=10.0 Euro delivery_start=2025-06-01 01:00:00 delivery_end=2025-06-01 02:00:00
```

is equivalent to

```
side=SELL volume=10.0 MW price=10.0 Euro delivery_start=2025-06-01 01:00:00 delivery_end=2025-06-01 01:15:00
side=SELL volume=10.0 MW price=10.0 Euro delivery_start=2025-06-01 01:15:00 delivery_end=2025-06-01 01:30:00
side=SELL volume=10.0 MW price=10.0 Euro delivery_start=2025-06-01 01:30:00 delivery_end=2025-06-01 01:45:00
side=SELL volume=10.0 MW price=10.0 Euro delivery_start=2025-06-01 01:45:00 delivery_end=2025-06-01 02:00:00
```

In real life, trades come in continuously, as they happen, through, for example, a websocket connection. To simplify
for this challenge, we will only look at an end-of-day file downloaded from the trading system.

### Revenue and PnL

The trading revenues are calculated by multiplying the volume with the price over all trades: for a buy trade,
the revenue is negative, for a sell trade, the revenue is positive.
The pnl is defined as the difference between the income made selling energy and the cost made buying it for a given
quarter-hour.
If we sell energy, our income is `quantity * price` since we got money for our electricity.
If we buy energy, our income is `-quantity * price`.
The PnL timeseries has a quarter-hourly resolution and for each quarter-hour, we add up the revenue for all trades for
which the delivery range contains the quarter-hourly timestamp.
For example, to compute the PnL for the timestamp `2025-06-01T10:15:00Z`, we consider (quarter-hourly) trades having
the delivery range `2025-06-01T10:15:00Z` to `2025-06-01T10:30:00Z` and (hourly) trades having the delivery range
`2025-06-01T10:00:00Z` to `2025-06-01T11:00:00Z`.

### Public Trades

The exchange also provides list of trades that were executed by all exchange participants, also called **public trades
**.
As for private trades, public trades would also be coming in continuously through a websocket connection but for this
challenge we are only looking at an end-of-day export, provided by the exchange as a csv file.

### VWAPs

Given a list of trades (public or private), we can define a **volume weighted average price** (also called VWAP)
quarter-hourly timeseries, as the sum of the revenue divided by the buy plus sell volume.\
For example, to compute the VWAP for the timestamp `2025-06-01T10:15:00Z`, we consider (quarter-hourly) trades having
the delivery range `2025-06-01T10:15:00Z` to `2025-06-01T10:30:00Z` and (hourly) trades having the delivery range
`2025-06-01T10:00:00Z` to `2025-06-01T11:00:00Z`.

We can calculate this quantity based on the private trades and it can be understood as a measure of the performance of
our trading if we compare it to the VWAP computed based on the public trades.
This is one example of a market **index** that can be built, but there are many more that give the traders insights on
price movements and feedback on their trading decisions.

## Invoicing

In the invoices we send out to the customers, we need to compute the following entries:

- **infeed payout**: multiplying the production of the asset by the price we agreed upon with the asset owner,
  depending on the price model.
  For the **fixed** pricing model, we just multiply the total produced volume with the price from the base data.
  This is a more "conservative" pricing model.
  For the **market** pricing model, we multiply the production of the asset, for each quarter-hour, with the market
  VWAP (VWAP computed with public trades).
  This is a price model that is more sensitive to the market price fluctuations and can yield a better payout if the
  asset is producing at times when the price on the market is high.
- **fees**: multiplying the production of the asset by the fee we agreed upon with the asset owner, depending on the fee
  model.
  For the **fixed_as_produced** fee model, we multiply the total produced volume with the fee from the base data.
  For the **fixed_for_capacity** fee model, we multiply the installed capacity of the asset with the fee from the base
  data.
  For the **percent_of_market** fee model, we multiply the total produced volume of the asset with the market VWAP and
  the fee percent from the base data.
  For each of these entries we also compute the unitary net amount, the VAT (at a rate of 19%) and the total gross
  amount.
- **redispatch payout**: We have to compensate customers for the redispatched volumes,
  as it was not "their fault" that their asset was turned off.
  The compensation is calculated by multiplying the latest forecasted volume for the redispatched asset with the
  compensation price provided by the DSO. Later on, FlexPower would claim this amount from the DSO by sending them
  an invoice.

## Imbalance

The electricity grid functions properly only when the production and consumption are balanced at all times, and
there are financial incentives to achieve that.

If our forecasted production is different from the measured production (higher or lower),
we have an imbalance volume. For that we have to pay a penalty.
The imbalance penalty is calculated in euro per megawatt-hour, we can then compute the imbalance cost for each asset
in our portfolio.
The dataset contains an estimated imbalance price and a final imbalance price. The imbalance penalty is computed using
the actual price if available, otherwise using the estimate.
For simplification, we provide the actual and estimated data as an end-of-day csv file. Some of the attributes in the
files are in German, as it is, in both cases, official data from the German regulator.

## The challenge

Your goal is to help FlexPower make sense of all this data and perform some key functions of the business.

As a general instruction, data needs to be ingested and persisted so that we can compute relevant
quantities and display them to the different users and teams.
Most data is coming from files that are distributed within directories, but this is rather a simplification.
In practice, data would come from APIs, SFTP servers, S3 buckets...

In particular, the dataset is centered around the delivery day 2025-06-08. Here is a list of aggregates that could be
built and visualized:

- compute the latest forecast for each asset and then the latest forecast for the whole portfolio.
- compute best-of-infeed on the asset level and then as an aggregation for the whole portfolio.
- compute the trading revenues, number of trades, net traded volume and VWAP.
- compute the imbalance cost for each asset and for the total portfolio.
- compute invoices for each asset.
- create a report that helps FlexPower understand the performance of its portfolio and each single asset.
- think about a permissions concept for data and reports.

This is rather a list to guide you through some interesting steps. There is no need to do all items in the list.
You are more than welcome to explore aspects that are not in this list.
In other words, the deliverable is left vague on purpose to give the freedom to explore, but if more guidance is needed,
we can provide more specific ways to tackle the problem.

## Submission Instructions

- Create a GitHub repository containing your solution and provide us with access so that we can review your code.
- Make sure to add all the necessary instructions to run the solution within the readme.
- Feel free to add any notes about technology choices and design decisions, as well as anything that we should keep in
  mind when reviewing your code. The challenge provides a lot of details
- AI is part of the developer tools now, so there is no problem leveraging it to help you solve the problem.
  If you do use AI, please provide us with a quick description of what tools you use and how you used them to work
  through the challenge.

## Notes

- Spend as much time as you want on the challenge to produce something you are proud of.
- The challenge is extensive, and you don't have to do everything, prefer quality to quantity. Feel free to make
  approximations (we do as well).
- You can use any tools you want, but we recommend using Python for any coding involved and SQL for queries and data
  processing.
- The solution is important, but so is everything else around it: unit tests, commit size and description, README and
  instructions to use the solution...
- If you already have similar personal projects that you would like to submit instead of this one, it's also possible.
  Just make sure that you include exhaustive documentation on what it's about, how we can run it and how it is relevant
  or similar to our challenge.

## A small comment on units
- volumes can be in megawatt (or MW, i.e. power) or megawatt-hour (or mwh, i.e. energy)
- converting from mw to mwh depends on the delivery range: if the delivery range is hourly, then the multiplication factor is 1, otherwise it is quarter-hourly, then the multiplication factor is 0.25, i.e.
```python
volume_mwh = volume_mw * 0.25
```
