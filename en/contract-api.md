<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Contract Interface](#contract-interface)
  - [Changelog](#changelog)
- [1. General Information](#1-general-information)
  - [1.1 Entry Point](#11-entry-point)
  - [1.2 Scale](#12-scale)
- [2 ABIs](#2-abis)
  - [2.1 ITrading](#21-itrading)
    - [2.1.1 Place Order](#211-place-order)
    - [2.1.2 Cancel Order](#212-cancel-order)
    - [2.1.3 Get Order](#213-get-order)
  - [2.2 IMarket](#22-imarket)
    - [2.2.1 Get Market Config](#221-get-market-config)
    - [2.2.2 Get Stock Config](#222-get-stock-config)
  - [2.3 IRiskControl](#23-iriskcontrol)
    - [2.3.1 Get Permissions](#231-get-permissions)
- [3 Reference Table](#3-reference-table)
  - [3.1 Order State](#31-order-state)
  - [3.2 Actions](#32-actions)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Contract Interface

## Changelog
- April 3, 2026
    - `sessionType` now supports `PRE_MARKET_AND_AFTER_HOURS` for LIMIT orders only
- March 10, 2026 
    - Support Market Order
- January 14, 2026
    - Initial release

# 1. General Information

The contract implements the [EIP-2535 Diamond](https://eips.ethereum.org/EIPS/eip-2535) standard, where a single contract address serves as the entry point.
Interacting with different facets requires their respective ABI interfaces.

## 1.1 Entry Point

**BSC**

- Testnet: [0x6c5A81eC1D8cF4A389F6Cc9498A3096CF823cb88](https://testnet.bscscan.com/address/0x6c5A81eC1D8cF4A389F6Cc9498A3096CF823cb88)
- Mainnet: [0xEfABC3ecb17232f27f6Be8AC16D2E423156A051D](https://bscscan.com/address/0xEfABC3ecb17232f27f6Be8AC16D2E423156A051D)

## 1.2 Scale

- For all values except the native token of the chain, use `6` decimal places.
- **Stable Tokens**: Must be uniformly adjusted to 6 decimal places. (Example: USDT has 18 decimals, so passing `0.01` USDT should be `10000`, not
  `10000000000000000`).
- **Native Tokens**: Follows the chain's native decimals.

# 2 ABIs

## 2.1 ITrading

* File: [ITrading.json](../abi/ITrading.json)

### 2.1.1 Place Order

`ITrading.placeOrder(NewOrder calldata order) external payable`

* NewOrder

| Parameter    | Type    | Required | Scale | Description                                                                          |
|--------------|---------|----------|-------|--------------------------------------------------------------------------------------|
| stockId      | uint32  | Y        |       | Stock ID                                                                             |
| tradeType    | uint8   | Y        |       | Order Type: 0-LIMIT 1-MARKET                                                      |
| side         | uint8   | Y        |       | Side: 0-BUY 1-SELL                                                                   |
| tif          | uint8   | Y        |       | Time-in-Force: 0-DAY _1-GTD_ _2-GTC_                                                 |
| sessionType  | uint8   | Y        |       | Trading Session: 0-DEFAULT, 4-PRE_MARKET_AND_AFTER_HOURS (LIMIT orders only)                                                           |
| paymentToken | address | Y        |       | Payment token (stablecoin) used for order collateral and settlement                  |
| validDate    | uint32  | Y        |       | Valid days when TIF is GTD, range: [1,30]; otherwise set to 0                        |
| networkFee   | uint64  | Y        | 6     | Set to 0 when paying with native tokens. Cannot be used together with native tokens. |
| amount       | uint64  | Y        | 6     | Set to 0 (Not used for size-based market orders)                         |
| price        | uint48  | Y        | 6     | For MARKET orders: Worst acceptable price (Slippage Protection). Set to `Current Price * (1 + slippage)` for BUY, or `Current Price * (1 - slippage)` for SELL                       |
| size         | uint48  | Y        | 6     | Order size. Must be a whole number of shares                                                     |

**Note:**

- The network fee paid in native token needs to have `value` set in the transaction.
- Network fee payment in stablecoins is not yet supported; `networkFee` should be set to `0`.
- `sessionType` currently accepts only `DEFAULT` and `PRE_MARKET_AND_AFTER_HOURS`. `PRE_MARKET_AND_AFTER_HOURS` can only be used with `LIMIT` orders.
- Italicized values are not currently supported.

**Example:**

**Limit Order**

```cgo
const limitOrder: ITrading.NewOrder = {
    stockId: 1,
    tradeType: TradeType.LIMIT,
    side: Side.SELL,
    tif: Tif.DAY,
    sessionType: SessionType.DEFAULT, // Set SessionType.PRE_MARKET_AND_AFTER_HOURS for pre-market and after-hours LIMIT orders
    paymentToken: "0x....",           // USDT address
    validDate: 0n,
    networkFee: 0n,
    amount: 0n,
    price: parseUnits("15", 6),
    size: parseUnits("5", 6),
};

ITrading.placeOrder(limitOrder, {value: 33000000000000})
```

**Market Order**

```cgo
const marketOrder: ITrading.NewOrder = {
    stockId: 1,
    tradeType: TradeType.MARKET,
    side: Side.BUY,
    tif: Tif.DAY,
    sessionType: SessionType.DEFAULT,
    paymentToken: "0x....",        // USDT address
    validDate: 0,
    networkFee: 0n,
    amount: 0n,
    price: parseUnits("15.15", 6), // 15 * (1 + 0.01) Slippage protection (1%)
    size: parseUnits("5", 6),      // Buy 5 shares
};

ITrading.placeOrder(marketOrder, {value: 33000000000000})
```

### 2.1.2 Cancel Order

`ITrading.cancelOrder(uint64 orderId) external`

**Example:**

```cgo
ITrading.cancelOrder(1938972632992);
```

### 2.1.3 Get Order

`ITrading.getOrder(uint64 orderId) returns(ITrading.OrderInfo)`

**Note:** 
Can only query open orders.

* ITrading.OrderInfo

| Parameter	        | Type      | 	Scale | 	Description                             |
|-------------------|-----------|--------|------------------------------------------|
| account           | 	address	 | 	      | The account that placed the order        |
| stockId           | 	uint32	  | 	      | Stock ID                                 |
| amount            | 	uint64	  |        | 	Frozen/collateral amount                |
| paymentToken	     | address   | 	      | 	Payment token                           |
| size              | 	uint48	  |        | 	Order size          |
| price	            | uint48		  |        | Worst acceptable price for market orders |
| tradeType         | 	uint8    | 	      | 	Order Type: 0-LIMIT 1-MARKET            |
| side	             | uint8	    |        | 	Side: 0-BUY 1-SELL                      |
| state             | 	uint8	  | 	      | [Order state](#31-Order-State)           |
| filledAmount	     | uint64	   | 6	     | Filled/collateral-released amount        |
| filledSize        | 	uint48   | 	6	    | Filled quantity                          |
| chargedFees       | 	uint40	  | 6	     | Platform fees already charged            |
| chargedCommission | 	uint40	  | 6      | 	Broker commission already charged       |

**Example:**

```cgo
ITrading.getOrder(123456784567);
// Returns
{
    "account": "0xabcd...",
    "stockId": 1,
    "amount": 10000000,        // 10.000000
    "paymentToken": "0x1234...",
    "size": 1000000,           // 1.000000
    "price": 10000000,         // 10.000000
    "tradeType": 0,
    "side": 0,
    "state": 0,
    "filledAmount": 900000,    // 0.900000
    "filledSize": 1000000,     // 1.000000
    "chargedFees": 6780,       // 0.006780
    "chargedCommission": 0,
}
```

## 2.2 IMarket

* File: [IMarket.json](../abi/IMarket.json)

### 2.2.1 Get Market Config

`IMarket.getMarketConfig() external view returns(IMarket.MarketConfig)`

**IMarket.MarketConfig**

| Parameter	             | Type	     | Required | 	Scale   | 	Description                                                                                                  | 
|------------------------|-----------|----------|----------|---------------------------------------------------------------------------------------------------------------|
| actions	               | uint64    | 	Y       | 	        | 	Allowed actions, represented as bits from right to left (bit set = allowed). refer to [Actions](#32-actions) |
| commissionRate         | uint32	   | Y	       | 6	       | Broker commission rate (fee calculated per share)                                                             |
| maxCommissionRate      | 	uint32	  | Y	       | 6	       | Maximum broker commission rate per order (fee calculated on trade amount)                                     |
| minCommissionPerOrder	 | uint32	   | Y        | 	6       | 	Minimum broker commission per order                                                                          |
| activityFeeRate          | 	uint32   | 	Y	      | 6	       | Trading activity fee rate (charged only on SELL)                                                              |
| minActivityFeePerOrder   | 	uint32   | 	Y       | 	6       | 	Minimum trading activity fee per order (charged only on SELL)                                                |
| maxActivityFeePerOrder   | 	uint32	  | Y	       | 6        | 	Maximum trading activity fee per order (charged only on SELL)                                                |
| networkFeeInNative     | 	uint128	 | Y	       | By chain | 	Minimum network fee when paying with native tokens                                                           |
| networkFeeInStable	    | uint32    | 	Y       | 	6       | 	Minimum network fee when paying with stablecoins                                                             |
| minAmountPerOrder      | 	uint32   | 	Y	      | 6	       | Minimum order amount                                                                                          |
| priceTTL               | 	uint32	  | Y	       |          | 	Oracle price validity period (seconds)                                                                       |
| priceTolerate	         | uint32	   | Y	       | 6	       | Maximum allowable deviation percentage between settlement price and oracle price                              |
| marketPriceTolerate    | 	uint32	  | Y	       | 6	       | Maximum allowable deviation percentage between settlement price and oracle price for market orders                                                      |

**Example:**

```cgo
IMarket.getMarketConfig();
// Returns
{
    "actions": 3,                           // Both buying and selling are allowed
    "commissionRate": 3500,                 // 0.0035
    "maxCommissionRate": 10000,             // 0.01
    "minCommissionPerOrder": 350000,        // 0.35
    "activityFeeRate": 166,                 // 0.000166
    "minActivityFeePerOrder": 10000,        // 0.01
    "maxActivityFeePerOrder": 8300000,      // 8.3
    "networkFeeInNative": 2500000000000,    // 0.0000025 (assuming 18 decimals)
    "networkFeeInStable": 2000,             // 0.002000
    "minAmountPerOrder": 20000000,          // 20.000000
    "priceTTL": 60,                         // 60 seconds
    "priceTolerate": 10000,                 // 0.01 (1%)
    "marketPriceTolerate": 50000            // 0.05 (5%)
}
```

### 2.2.2 Get Stock Config

`IMarket.getStockConfig(uint32 stockId) external returns (IMarket.StockConfig)`

**IMarket.StockConfig**

| Parameter	 | Type	    | Required | 	Scale	 | Description                                                                                                   |
|------------|----------|----------|---------|---------------------------------------------------------------------------------------------------------------| 
| token	     | address	 | Y	       |         | 	Contract address of the RWA asset                                                                            |
| actions    | 	uint64	 | Y	       |         | 	Allowed actions, represented as bits from right to left (bit set = allowed). refer to [Actions](#32-actions) |
| feeRate    | 	uint32	 | Y        | 	6	     | Platform fee rate (fee calculated on trade amount)                                                            |

**Example:**

```cgo
IMarket.getStockConfig(1);
// Returns
{
    "token": "0x...",
    "actions": 3,
    "feeRate": 4000 // 0.004
}
```

## 2.3 IRiskControl

* File: [IRiskControl.json](../abi/IRiskControl.json)

### 2.3.1 Get Permissions

`IRiskControl.getPermissions(address account) external returns(uint)`

For return values, refer to [Actions](#32-actions).

**Example:**

```cgo
IRiskControl.getPermissions("0x...")
```

# 3 Reference Table

## 3.1 Order State

| State | Note             |
|-------|------------------|
| 0     | New              |
| 1     | Partially Filled |
| 2     | Failed           |
| 3     | Cancelled        |
| 5     | Filled           |
| 8     | Pending Cancel   |


## 3.2 Actions

| Bit | Value | Note           |
|-----|-------|----------------|
| 0   | 1     | Buy            |
| 1   | 2     | Sell           |

**Example:**
- 0: Neither buying nor selling is allowed.
- 1: Only buying is allowed.
- 2: Only selling is allowed.
- 3: Both buying and selling are allowed.
