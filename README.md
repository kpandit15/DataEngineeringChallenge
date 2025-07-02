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

One of FlexPower's main products is the so-called **RouteToMarket**: bringing the energy produced by renewable
assets to the European electricity markets.
FlexPower signs contracts with renewable asset owners, and then sells the energy produced by
these assets on the electricity market, hopefully profitably thanks to its trading expertise.

At the end of each month, FlexPower invoices the asset owners based on the production of their assets:
FlexPower pays the asset owner an agreed-upon price for each produced MWh, and the asset owner pays FlexPower
a fee for the services provided, per MWh.
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
- fee_percent: percentage of the market value paid by the asset owner to FlexPower per MWh, in %, if asset in the
  "percent_of_market" fee model.

A new file is created every week, on Sunday, between 04:30 and 05:00.
The portfolio operations team might create new files on demand if some urgent changes need to be ingested.

## Infeed(s)

Given how the RouteToMarket product functions, infeed data plays a central role in our system.
There are different types of infeed data, with different functions and degrees of trustworthiness.

### Forecasts:

For each asset, FlexPower forecasts the electricity production for 15-minute delivery intervals,
representing the average power expected to be produced by the asset over a quarter-hour, in kilowatt (kW).\
This forecast is updated every 15 minutes and allows FlexPower to know how much electricity will be available to sell
on the markets for a quarter-hour in the future. The latest forecast for each quarter-hour is calculated at the start
of each 15-minute delivery range, but the latest "tradeable" forecast for each asset is produced 15 minutes before
delivery start (due to trading gate closures).\
For example, the latest forecast for the delivery start `2025-06-08T12:00:00Z` has the version `2025-06-08T12:00:00Z`
but the latest tradeable forecast has the version `2025-06-08T11:45:00Z`.\
To query the forecasts, use the following code snippet: