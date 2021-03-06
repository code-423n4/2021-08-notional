# Notional V2 Whitepaper

Authors: Teddy Woodward <teddy@notional.finance>; Jeff Wu <jeff@notional.finance>

Notional brings fixed rate, fixed term lending and borrowing to Ethereum. In Notional V1, we introduced many important concepts such as cash balances and fCash, the Notional portfolio for enabling cross-margin trading, the Notional AMM and the settlement of matured assets.

In Notional V2, our focus is to further refine these concepts and optimize for greater efficiency, scalability and longevity. In this whitepaper we will discuss the major improvements in Notional V2 which include convertible fCash, flash loan resistant AMM improvements, a perpetual liquidity token, gas efficient and scalable idiosyncratic fCash, and Cash Group improvements to allow for the trading of 5 year, 10 year and longer-dated fCash assets.

Furthermore, we will also discuss general technical architecture improvements that will make future development and upgrades to Notional V2 more robust.

For a review of Notional concepts, please refer to the [Notional V1 Whitepaper](https://docs.notional.finance/developers/whitepaper/whitepaper).

A less technical guide to Notional V2 is available [here](https://docs.notional.finance/notional-v2).

## Liquidity Curve Changes

The basics of the liquidity curve are not changing from Notional V1, but we are introducing three new concepts to the existing curve. All details of the new liquidity curve are included in [Appendix A](#appendix-a).

### Convertible fCash

Convertible fCash is an extension of fCash. Convertible fCash is denominated in one currency (ex. Dai) but settled in a different currency (ex. cDai). Broadly speaking, convertible fCash represents an amount of the denomination currency (Dai) in terms of the settlement currency (cDai) at its maturity. For example, 100 January 1st 2022 convertible fDai (simply called fDai from here on) can be thought of as 100 Dai’s worth of cDai as of January 1st 2022. On January 1st 2022, 100 fDai entitles the holder to an amount of cDai that is equivalent to 100 Dai using the Dai/cDai exchange rate as of maturity as a reference.

Convertible fCash enables liquidity providers to earn an interest rate on their capital while they provide liquidity because they can put cTokens into Notional's liquidity pools instead of the underlying token which bears no interest. This is important to maximize capital efficiency for liquidity providers. An example illustrates how this works:

A liquidity provider deposits cDai (Compound Dai Tokens) instead of Dai into Notional's liquidity pools. This cDai is a claim on the interest paid by Compound Dai borrowers and accumulated every block. Over time, the cash side of the liquidity pool will passively earn Compound interest which will accrue to the liquidity providers' cash claims.

It's important to note that convertible fCash markets still enable the lending and borrowing of the underlying token (Dai, not cDai). If a lender wants to lend Dai to this market, Notional will wrap their Dai into cDai, place it in the market and give the lender fDai in return. The fDai that the lender receives will represent a fixed amount of Dai at maturity. Similarly, a borrower will receive the cDai equivalent of the amount of Dai that they borrowed and their negative fDai position will represent the fixed amount of Dai they owe at maturity. Notional will wrap and unwrap cDai depending on the user's preference.

#### Convertible fCash Settlement

Upon settlement, the amount of fDai in a user's portfolio (representing a fixed amount of Dai) will be converted into the equivalent amount of cDai at maturity. For any given maturity, all fCash of that maturity will convert to cDai at the same rate ensuring that the net fCash balance of the system in cDai terms always sums to zero.

For example, with a settlement cDai/Dai rate of 10:1 (10 cDai per 1 Dai), 100 fDai will convert into 1000 cDai at maturity.

Setting this settlement rate will be done by the first account to settle an asset at each maturity. Assets can be settled by anyone after they mature. Because Compound and other variable rate instruments accrue interest on a per block basis, these rates are not vulnerable to flash loan type manipulation.

#### Convertible fCash Benefits

The benefits are quite clear for liquidity providers, their capital is now earning interest on an underlying risk free rate in addition to the liquidity fees generated by Notional trading. High returns will attract more liquidity and result in better pricing for lenders and borrowers. Additionally, lenders will accrue passive interest on their cash balances after maturity. Borrowers will similarly accrue interest on their debts after maturity.

### On-Chain Rate Oracle

In order to maximize capital efficiency for borrowers, all fCash assets in Notional are automatically used as collateral for debts. Notional liquidity pools, however, could be vulnerable to flash loan manipulation like all AMMs. This is why Notional V1 does not rely on its own liquidity pools to set the value of fCash assets. Notional V1 instead relies on a governance parameter which values fCash receiver assets at a 50% annualized discount. Although this is a very safe assumption, it is also a significant discount from the fCash's actual value.

In Notional V2 we introduce an on-chain rate oracle that can be used to value fCash. It uses a lagged, weighted average of the interest rate which dampens the effect of flash loans. The formula is as follows:

```
oracleRate =
  annualizedRatePreTrade *
    min((currentBlockTime - previousBlockTime) / timeWindow, 1) +
  oracleRatePrevious *
    max(timeWindow - (currentBlockTime - previousBlockTime) / timeWindow, 0)
```

This ensures that the last rate the market traded at is averaged into the oracle price over a time window. Flash loan transactions within a single block will have a `currentBlockTime - previousBlockTime` value of `0` which means they will have no effect on the oracle interest rate. A non-flash loan trade that significantly moves markets will be averaged into the oracle price over the time window which may be set on the order of minutes to hours.

#### Benefits

This rate oracle allows us to mark the value of fCash to on chain markets and enables many of the new features introduced in Notional V2 including [Settlment via Debt](#settlement-via-debt) and [Idiosyncratic fCash](#idiosyncratic-fcash)

### Continuous Compounding

One of the goals of Notional V2 is to enable fCash assets with 20 year maturities. Currently we use simple interest rates which will become inaccurate over longer time horizons. Notional V2 will use continuous compounding instead and the change is described in [Appendix A](#appendix-a).

## Settlement via Debt

Matured fCash debt must be settled to ensure that there is sufficient underlying currency to allow lenders to withdraw. Notional V1 sells a debtor's collateral at a penalty to recapitalize cash balances. This, however, is quite painful for borrowers as they lose their collateral. In Notional V2, we settle negative cash balances by allowing a lender to deposit cash and lend directly to the borrower at a penalty rate relative to the [On-Chain Rate Oracle](#on-chain-rate-oracle) at the 3 month maturity.

This is the equivalent of the borrower "rolling their borrow" forward 3 months at a time at a new fixed interest rate. The cash the lender deposits will recapitalize the system. If the borrower ever becomes undercollateralized as a result of the interest they accrue they can be liquidated.

## Cash Group Improvements

In Notional V1, the concept of a cash group represents a recurring cadence of liquidity pools parameterized by a currency, a maturity length and the number of maturities. For example, a cash group of Dai with two 3 month maturities will have two liquidity pools at 3 months and 6 months.

This presents problems. If we want to have two 3 month maturities and a one month maturity for example, we would need two different cash groups - one 3-month cash group with two maturities and one 1-month cash group with one maturity. Every third month we would have an overlap - the 1 month maturity would coincide with one of the 3-month maturities. Handling this is awkward at best and would likely result in the fragmenting of liquidity and a poor UX.

Another problem with the V1 cash group structure is that long-dated maturities have correspondingly long settlement cycles which means that users can't reliably borrow and lend to a given tenor. Consider a five year maturity. In one year's time, that would become a four year maturity and a five year maturity would no longer be available. And then after another year, a three year maturity, and so on. This would mean that a user would only be able to lend or borrow for a five year period once every five years.

### Persistent Maturities

We resolve this in Notional V2 using persistent maturities. Each cash group has a set maturity cadence, for example: 3 Month, 6 Month, 1 Year, 2 Year, 5 Year. Instead of only opening up a new maturity when an existing maturity rolls down to t0, we roll all maturities every quarter and re-initialize new liquidity pools with the set maturity cadence calculated from the reference time of that quarterly roll. Every quarter, Notional will settle liquidity tokens in all markets and launch new markets at their target maturities. The [nToken](#perpetual-liquidity) will be used to ensure that markets have a baseline amount of liquidity at initialization.

Take the following cadence for example:
fCash: 3 month, 6 month, 1 year, 2 year, 5 year initialized on Jan 1 2021

On Apr 1 2021, the following will happen:

- 3 month fCash and liquidity tokens will mature to cash
- 6 month fCash will become tradable in the new 3 month fCash market
- 1 year fCash will now become 9 month [idiosyncratic fCash](#idiosyncratic-fcash) maturing on Jan 1 2022
- 2 year and 5 year fCash will become similarly idiosyncratic
- 1, 2 and 5 year liquidity tokens will settle to cash and idiosyncratic fCash
- New 6 month, 1 year, 2 year and 5 year markets will be initialized via [nTokens](#perpetual-liquidity). These new markets will mature on Oct 1 2021, Apr 1 2022, Apr 1 2023, Apr 1 2026 respectively.
- Previously idiosyncratic fCash maturing on Oct 1 2021 (6 month fCash) will now be liquid on the new 6 month fCash market. Similarly for other idiosyncratic fCash that now correspond to an active market.

The key benefit of this structure is to ensure that users can always trade fCash at approximately 1 year, 2 year or other longer dated maturities. In the Notional V1 cash group structure, we would have to choose between fragmenting liquidity or long time gaps between long dated fCash markets.

The downside is that longer dated fCash becomes less liquid when it changes to idiosyncratic fCash. We believe this is a necessary and reasonable trade off. Holders of 4.5 year fCash can resort to OTC markets to exit their position if necessary. We describe how we enable efficient, on-chain OTC markets in Notional V2 in a later section.

## nTokens

Providing liquidity in Notional V1 requires some amount of active participation. Liquidity tokens mature with the market and must be periodically rolled to new maturities. Additionally, the initialization of fCash markets poses a significant user experience issue. New fCash markets require a liquidity provider to set the market rate before they can be traded. This is inconvenient for liquidity providers who want passive exposure to trading on all fCash maturities.

It also requires a knowledgeable liquidity provider to set the initial rate. This is inconvenient for lenders and borrowers who are prevented from trading on the market until a liquidity provider initializes the market.

Notional V2 creates a perpetual liquidity token (called the `nToken`) to resolve this issue. Each currency type that Notional supports for lending/borrowing will have its own nToken (nDai, nUSDC, nETH, etc.). The nToken is an ERC20 token that represents a liquidity provider's share of a passive pool of liquidity provided to all active maturities of the given currency type at ratios determined by governance. Notional V2 retains the ability to add and remove liquidity from individual markets so that active liquidity providers can respond to market demand.

### Adding Liquidity

A liquidity provider deposits cash (i.e. cDai) into the nToken which is then provisioned across all fDai cash markets at predetermined ratios. For example, if there are 3 fCash markets (3 month, 6 month, 1 year) and governance has set predetermined ratios of (20%, 40%, 40%) then a 100 cDai deposit will be split into each market at the respective percentage.

The liquidity provider is then issued a perpetual, non-maturing ERC20 token (the nToken) that represents their share of all the cash and fCash assets in the nToken portfolio. This token can be used as collateral on Notional V2 and potentially other protocols. Because its underlying assets are a basket of cDai, fDai and the future cash flows from trading fees in all fDai markets, we believe that it will be an attractive option for high-quality, yield-generating collateral.

This ERC20 token is modeled as a special portfolio in Notional V2 that holds all the corresponding liquidity tokens.

### Automatic Market Initialization

The nToken's assets can be used to initialize markets every 90 days. This ensures that new markets will always have a source of liquidity and it allows Notional V2 to ensure that markets are properly initialized. Market initialization is done in a way that ensures newly created markets have oracle rates that align with the previous quarter's oracle rates. This ensures that on chain fCash values do not experience a sudden change when new markets are initialized.

There is one scenario to keep in mind when it comes to market initialization. The nToken does not provide liquidity above a certain market proportion of cash to fCash (more on this in [Appendix D](#appendix-d)). If maintaining oracle rates requires a market proportion above the leverage threshold, the nToken will initialize the market at the leverage threshold and the oracle rate will shift.

More details on market initialization method are described in [Appendix F](#appendix-f)

### Settlement

The dynamics of liquidity tokens are unchanged from Notional V1. A liquidity provider will lend to a borrower or borrow from a lender at the market rate, taking a portion of the slippage as their fee. This fee will accrue as both cash and fCash. Liquidity providers provide liquidity with leverage, so their claim on the fCash receivers in the market are partially offset by an fCash payer position in their portfolio. Therefore, when these tokens are settled, liquidity providers are left with some amount of cash and net fCash.

In Notional V1, liquidity tokens mature at the same cadence as the fCash position. This means that a liquidity token in a 1 year fCash market will mature in 1 year along with the corresponding fCash. However, in Notional V2 we have [persistent maturities](#persistent-maturities) instead. In this model, the liquidity token for the 1 year fCash market will settle at the end of a 3 month trading period while the 1 year fCash will mature 9 months later.

This means that the nToken will likely hold residual, [idiosyncratic fCash](#idiosyncratic-fcash) positions that will mature over time. This is due to the fact that a liquidity provider may end up as a net lender or borrower as a result of market activity.

Governance must carefully monitor the fCash balances for the nToken because large future fCash balances imply less cash to initialize new markets. Due to the nature of [persistent maturities](#persistent-maturities), however, governors will have an opportunity to modify market parameters to adjust for potential imbalances every quarter.

These idiosyncratic fCash balances will become a drain on the total liquidity available to the nToken over time if left unmanaged. Governance will set a discount rate for idiosyncratic fCash in order to recapitalize liquidity from these residual fCash balances.

If the liquidity provider is left a net borrower after a market settles, it will reserve sufficient cash to purchase offsetting fCash assets to negate its idiosyncratic balances. The reason for this is that each idiosyncratic asset increases the gas cost of redeeming and valuing nTokens.

### Redeeming nTokens

Redeeming nTokens means taking a portion of the cash and **all** [idiosyncratic fCash](#idiosyncratic-fcash) positions that the token contract holds. If a cash group is trading in 3 month, 6 month, 1 year, 2 year, 5 year, 10+ year markets as we intend for Notional V2, the gas costs of performing this action may become quite large except for [idiosyncratic market makers](#idiosyncratic-market-maker).

Setting aggressive prices to ensure that nToken accounts sell their residual idiosyncratic fCash will reduce this overhead to a manageable level. Also, the technical architecture of Notional V2 will allow users to withdraw and sell back residual non-idiosyncratic fCash in a single transaction, greatly reducing the gas cost of withdrawing liquidity. 

### Liquidation

The nToken must never become undercollateralized. This can only happen if liquidity is provided at a leverage ratio that exceeds the potential for future losses. This ratio is a function of a given market's time to maturity. Governance parameters will ensure that markets are never initialized at an unsafe leverage ratio. If additional liquidity must be added to a market with an unsafe leverage ratio, the nToken will lend to the market (purchase fCash) instead of providing liquidity until the leverage ratio is reduced to an appropriate level.

We discuss this further in [Appendix D](#appendix-d).

### Benefits

To summarize, nTokens allow users to passively provide liquidity across all fCash markets without the need to roll their liquidity or interact with the individual maturities in any way. In return, these users are issued a perpetual ERC20 token that can be freely exchanged and used as collateral in existing DeFi infrastructure. In addition, nTokens ensure all Notional V2 markets have sufficient liquidity when they become active every 90 days.

## Idiosyncratic fCash (ifCash)

Idiosyncratic fCash refers to fCash that does not fall on a maturity with an active market. So if the current active maturities are April 1st and July 1st, fCash maturing on any other date is idiosyncratic. Notional V2 supports scalable idiosyncratic fCash - users can trade fCash at any reasonable future date up to 20 years and potentially beyond. This will bring DeFi much closer to feature parity with traditional financial systems.

The collateral requirement of a large idiosyncratic fCash portfolio must take into account the **net** value of all its fCash positions so market makers can hold an efficient amount of collateral against a portfolio with large, but offsetting, fCash balances. In effect, this means that positive idiosyncratic fCash positions must count as collateral, just like non-idiosyncratic fCash.

Finally, liquidation of large portfolios must remain efficient in order to ensure the integrity of the system as a whole.

### Market Maker Portfolios

Most users of Notional V2 will have a portfolio very similar to the one that exists in Notional V1. This portfolio is simply an array of assets limited by a maximum assets governance parameter. However, Notional V2 also introduces a second "asset bitmap" portfolio that can be enabled on any account and will be especially useful for market makers.

When using asset bitmaps, a portfolio can store ifCash at 256 predetermined dates in the future. This portfolio will contain:

- A 32 byte bitmap corresponding to the presence of ifCash at predetermined dates.
- A mapping between the maturity of an ifCash asset and its notional value.

### Predetermined Dates

The 32 byte bitmap (256 bits) is structured with four time chunks that ensure ifCash will periodically align with on chain markets, allowing the account to sell off the position on chain if required.

Each bit references a date relative to the current block time at UTC midnight:

- Bits 1 to 90 represent 1 day offsets
- Bits 91 to 135 represent 6 day offsets (Bits 1 to 135 cover 360 days)
- Bits 136 to 195 represent 30 day offsets (coverage up to 5 years)
- Bits 196 to 256 represent 90 day offsets (coverage up to 20.25 years)

ifCash can be traded up until the longest dated on chain market, which ensures that all ifCash assets can be valued relative do on chain rate oracles.

### Trading ifCash

Creating ifCash is as simple as initiating an ERC1155 asset transfer. A market maker can transfer 1000 fDai to a taker. The taker will receive 1000 fDai and the market maker will have 1000 fDai debited from their portfolio. If the balance becomes negative, the market maker simply needs to pass a free collateral check because they have become a net borrower. Because the positive and negative fCash balances offset, the system as a whole remains in balance. Any asset transfers between the maker and the taker must be done outside the system.

Market makers can also instruct the ERC1155 contract to issue a transaction on the Notional contract to deposit collateral or trade. This allows market makers to lend idiosyncratic via OTC markets and borrow via Notional liquidity curves to fulfill the trade in a single transaction. Conversely, market makers will be able to borrow idiosyncratic via OTC and lend the cash they've borrowed via Notional's on chain markets in a single transaction.

### ifCash Settlement

In a non market maker portfolio, the assets refer to absolute dates (i.e. Jan 1 2022). However, the ifCash bitmap refers to relative dates. At every transaction, Notional V2 must update the bitmap to ensure that it continues to reference the correct relative points in time.

A full description of the algorithm and proofs are provided in [Appendix B](#appendix-b).

### ifCash Valuation

Notional V2 will value all ifCash positions using the interest rates provided by the [On-Chain Rate Oracle](#on-chain-rate-oracle). We interpolate the set of on-chain rate oracles for each currency to create a yield curve. The discount rate used to value the ifCash is then taken from that interpolated yield curve.

Any ifCash asset's discount rate will be calculated as the interpolation of two liquid interest rates on chain. For assets maturing before the 3 month market (the shortest term market) the rates will be the per block interest rate (i.e. cToken supply rate) and the interest rate at the 3 month market.

During a free collateral check, Notional V2 will fetch all the relevant rates and discount each of the stored notional values to present value. This (along with some haircut) will be the net value of the ifCash in a market maker portfolio. The method is described in [Appendix C](#appendix-c).

### Liquidation

During liquidation, if the market maker portfolio has non-local currency collateral it can be purchased at a discount in exchange for local currency (similar to a regular liquidation).

If the market maker must be liquidated via fCash a liquidator can purchase any fCash or ifCash asset of their choosing at a discount to its present value as determined by the on chain oracle. There would be no difference in liquidating a market maker portfolio versus a normal portfolio.

## Upgradeable Architecture

Notional V2 incorporates many learnings from Notional V1 for improving the gas efficiency, upgradeability and testability of smart contracts. This section will briefly describe some of the technical improvements in Notional V2.

The core development principles in Notional V2 are as follows:

- Place all system state within a single storage trie
- Isolate logic for reading and writing to storage
- Isolate all business logic in internal libraries to allow for mocking
- Compose external method calls from a set of calls to internal libraries
- Minimize or eliminate inter-contract method calls

Standard interfaces such as ERC20 would be exposed via a proxy contract and state would still be retained on the Notional V2 proxy. The goal here is to limit inter-contract data copies which are gas inefficient.

A Notional V2 system would look as such on chain:

![](https://i.imgur.com/H3pZPK3.png)

## Appendix A

This appendix includes a summary of the liquidity curve as a result of the changes in Notional V2. Key differences include:

- Conversion of current assets to underlying value (i.e. cDai to Dai)
- Implied rates are stored as annualized rates

```syntax=python
def calculateTrade(market, fCashToAccount, cashGroup, timeToMaturity):
    # Convert cDai balances to DAI equivalents
    totalCashUnderlying = market.totalAssetCash.convertToUnderlying()

    # Calculate the rate anchor (offsets the logit curve on the y axis)
    newExchangeRate = math.exp(market.lastImpliedRate * timeToMaturity / SECONDS_IN_YEAR)
    proportion = market.totalfCash / (market.totalfCash + totalCashUnderlying)
    newRateAnchor = newExchangeRate - math.log(proportion / (1 - proportion)) / rateScalar

    # Calculate the exchange rate between fCash and cash
    exchangeProportion = (market.totalfCash - fCashToAccount) / (market.totalfCash + totalCashUnderlying)
    preFeeExchangeRate = math.log(exchangeProportion / (1 - exchangeProportion)) / rateScalar + newRateAnchor

    # Calculate fees to reserve and liquidity providers
    feeRate = math.exp(feeRateBasisPoints * timeToMaturity)
    if (lending) {
        postFeeExchangeRate = preFeeExchangeRate / feeRate
    } else if (borrowing) {
        postFeeExchangeRate = preFeeExchangeRate * feeRate
    }

    # Calculate cash amounts
    netCashToAccount = -(fCashToAccount / postFeeExchangeRate)
    netFee = math.abs((fCashToAccount / preFeeExchangeRate) - (fCashToAccount / postFeeExchangeRate))
    cashToReserve = netFee * reserveFeeShare
    netCashToMarket = -(netCashToAccount - fee + cashToReserve)

    # Update market state
    market.totalCurrentAssets += netCashToMarket.convertToAssetCash()
    market.totalfCash += fCash
    market.lastTradeTime = blockTime

    # Calculate the new implied rate for the market for the next trade
    exchangeProportion = (market.totalfCash - fCashToAccount) / (market.totalfCash - fCashToAccount + totalCashUnderlying + netCashToMarket)
    newMarketExchangeRate = math.log(exchangeProportion / (1 - exchangeProportion)) / rateScalar + newRateAnchor
    marketState.lastImpliedRate = math.log(newMarketExchangeRate) * SECONDS_IN_YEAR / timeToMaturity

    reserve += cashToReserve

    return netCashToAccount
```

## Appendix B

This appendix describes how an asset bitmap is updated during settlement.

These are the relevant data objects:

- 256 bit ifCash bitmap: each bit signifies the presence of ifCash at the corresponding date
- Next Settle Date: set to the UTC midnight date of the first bit
- mapping(maturity => int): a mapping between maturities and the notional ifCash value held at that time

The bitmap is defined as follows (1-indexed, inclusive):

Define `t = blockTime - (blockTime % 86400)` (current day, midnight utc)

- Bit 1 to 90: `t + bitNum * 1 day`
- Bit 91 to 135: `t + 90 days - (t mod 6 days) + (bitNum - 90) * 6 days`
- Bit 136 to 195: `t + 360 days - (t mod 30 days) + (bitNum - 135) * 30 days`
- Bit 196 to 256: `t + 2160 days - (t mod 90 days) + (bitNum - 195) * 90 days`

The max relative day for each block is:

- Max Day Block: `t + 90`
- Max Days of Week Block: `t + 90 - 0 + (135 - 90) * 6 = t + 360`
- Max Days of Month Block: `t + 360 - 0 + (195 - 135) * 30 = t + 2160`
- Max Days of Quarter Block: `t + 2160 - 0 + (256 - 195) * 90 = t + 7650`

#### From Maturity to Bit Number

```syntax=python
def getBitNumFromMaturity(nextMaturityDate, maturity):
  offset = maturity - nextMaturityDate
  t = nextMaturityDate / SECONDS_IN_DAY
  require(offset > 0)

  if offset <= 90:
    return offset
  if offset <= 360:
    return 90 + floor((n - 90) / 6)
  if offset <= 2160:
    return 135 + floor((n - 360) / 30)
  if offset <= 7650:
    return 195 + floor((n - 2160) / 90)
```

#### From Bit Number to Maturity

```syntax=python
def getMaturityFromBitNum(t, bitNum):
  if bitNum <= 90:
    return t + bitNum days
  if bitNum <= 135:
    return t + 90 days + (bitNum - 90) * 6 days
  if bitNum <= 195:
    return t + 360 days + (bitNum - 135) * 30 days
  if bitNum <= 256
    return t + 2160 days + (bitNum - 195) * 90 days
```

### Mapping on Roll Down

As assets roll down from higher time blocks to lower time blocks, we must ensure that they can be mapped to an existing bit number. Since all the time granularities are factors of each other, we know that the dates in higher time granularities will be contained in lower granularities.

If `ac = bc mod mc` then `a = b mod m`, therefore all of these are equal:

- `d = t mod 1`
- `d * 6 = t * 6 mod 6`
- `((d * 6) * 5) = ((t * 6) * 5) mod (6 * 5)`
- `d * 30 = t * 30 mod 30`
- `(d * 30) * 3 = (t * 30) * 3 mod (30 * 3)`
- `d * 90 = t * 90 mod 90`

## Appendix C

This section discusses the ifCash valuation. The continuous compounded present value is defined as `pv = v * e^(-rt)`.

The rate at an illiquid point, idiosyncratic point is defined as the linear interpolation between its nearest two liquid points. This is defined as: `r = (r1 - r0) * (offset / length) + r0` where `offset` is the time from `r0` and length is the time between `r0` and `r1`.

This function is the equivalent of:
`r = r1 * (offset / length) - r0 * (1 - offset / length)`

Substituting in we find:
`pv = v * e^(r0 * t * (1 - offset / length) - r1 * t * (offset / length))`

We calculate this present value for every ifCash asset in the portfolio.

### Current per Block Rates

When `r0` in the approximation above refers to the current time the rate we use will the the interest rate of asset that the market holds. If, for example, the market refers to a cToken `r0` will be the Compound per block supply rate converted into an annualized interest rate.

## Appendix D

This appendix discusses liquidity provider leverage and caps on leverage ratios that ensure the nToken will never be liquidated.

The nToken, like all liquidity providers in Notional, provides liquidity to individual markets with leverage. The higher the proportion of fCash to cash in the market at the time the LP (or nToken) provides liquidity, the higher leverage the LP gets. This leverage makes Notional more capital efficient, but it also introduces risk. Consider the following two examples:

1. An LP provides 100 cash of liquidity to a market with an equal amount of cash and fCash (a proportion of .5). In this scenario, the LP would have -100 fCash in their portfolio alongside liquidity tokens equivalent to +100 cash and +100 fCash.

2. An LP provides 100 cash of liquidity to a market that has much more fCash than cash, specifically 9x more fCash than cash (a proportion of .9). In this scenario, the LP would have -900 fCash in their portfolio alongside liquidity tokens equivalent to +100 cash and +900 fCash. 

As lenders and borrowers trade on these market, the LP's liquidity token claims will change. If the traders on this market are net lenders, the LP will develop a net short fCash position coincident with decreasing interest rates. This will result in what is called impermanent loss because the present value of their negative fCash balance increases with lower interest rates. The shorter the LP becomes, the lower interest rates go, and the more money they lose. The difference between scenario 1 and 2 is that in scenario 2 the LP will get short 9x faster, and thus has a 9x greater capacity for loss relative to the same +100 cash deposited as liquidity in scenario 1.

The free collateral calculation reflects the greater risk of the LP's position in scenario 2. Assuming a liquidity token haircut of 90%, and an fCash valuation discount factor of 1 for simplicity, the LP in scenario 1 will have free collateral of ~ +80. In scenario 2 though, the LP will have free collateral of ~ 0.

So the LP in scenario 2 not only starts out with less free collateral than the LP in scenario 1, their free collateral position will deteriorate 9x more quickly due to impermanent loss. Clearly then, the LP in scenario 2 is at risk of falling undercollateralized and being liquidated. 

We can never let this happen to the nToken. Luckily, we can ensure that this doesn't happen by capping the leverage ratio at which the nToken provides liquidity to any market. The exact placement of the cap depends on the liquidity token haircut, the fCash haircut, the duration of the maturity, the rateAnchor, and the rateScalar.

It's difficult to solve for the leverage cap analytically, but it is straightforward to solve for via simulation. In practice, leverage caps will be set in conjunction with simulation work demonstrating their effectiveness.

## Appendix E

Incentives are minted for nToken holders based on their share of total supply, an annual incentive emission rate parameter, and a multiplier for long term nToken holders. The formula is as follows:

```
multiplier = 1 + (proRataYears * multiplierConstant)
incentivesToClaim = (tokenBalance / avgTotalSupply) * emissionRatePerYear * proRataYears * multiplier`
avgTotalSupply = (currentTotalSupply + totalSupplyAtLastClaim) / 2
```

The multiplier is capped at 2. Because the total supply of nTokens fluctuates, Notional uses an average of the total supply at the time when the account minted nTokens and the current supply of nTokens when calculating incentives to claim.

## Appendix F

Every quarter, markets will be re-initialized and the on-chain rate oracles will migrate from the old set of maturities to the new set of maturities. Notional V2's aim here will be to keep portfolio valuations constant through this process. This means that the newly initialized rate oracles must be set at rates which keep the valuation curve unchanged. For example, consider an account with a 9 month fCash position that is valued directly from an on-chain oracle at an implied rate of 5% annualized. When the markets roll forward that 9 month position will now be idiosyncratic, and will be valued via linear interpolation between the new 6 month market and the new 1 year market. Notional V2 must ensure that the two new markets are initialized at rates which keep the 9 month rate constant at 5%.

Initializing markets keeps the valuation curve constant by proceeding in this fashion:

- Existing liquidity tokens are settled to cash and fCash positions.
- Cash is reserved to repay any negative fCash positions.
- The 6 month market is now the new 3 month market, it's liquidity does not change.
- A new 6 month market will be initialized with the interpolated rate between the new 3 month market and the old 1 year market (now with 9 months time to maturity).
- This is repeated for subsequent markets.

The formula for initialization is:

```
exchangeRate = e^(impliedRate * timeToMaturity)
exchangeRate = (1 / rateScalar) * ln(proportion / (1 - proportion)) + rateAnchor

rateScalar * (exchangeRate - rateAnchor) = ln(proportion / (1 - proportion))
e^(rateScalar (exchangeRate - rateAnchor)) = proportion / (1 - proportion)
proportion = exp / (1 - exp)

proportion = totalfCash / (totalfCash + totalCurrentCash)
totalCurrentCash = totalfCash * (1 - proportion)
totalfCash = totalCurrentCash / (1 - proportion)
```
