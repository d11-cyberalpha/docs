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
    - [2.2.3 Get Fee Config](#223-get-fee-config)
  - [2.3 IRiskControl](#23-iriskcontrol)
    - [2.3.1 Get Permissions](#231-get-permissions)
- [3 Reference Table](#3-reference-table)
  - [3.1 Order State](#31-order-state)
  - [3.2 Actions](#32-actions)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Contract Interface

## Changelog
- May 27, 2026
    - Support B-side contract access
    - Support configurable brokerage fee rules
- May 14, 2026
    - `sessionType` now supports `PRE_MARKET`, `AFTER_HOURS`, and `OVERNIGHT`; non-`DEFAULT` sessions are available for `LIMIT` orders only
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

- Unless otherwise specified, numeric values such as amounts, prices, and sizes use `6` decimal places.
- **Stable Tokens**: Values must be normalized to `6` decimal places before submission. For example, if USDT uses `18` decimals, `0.01` USDT must be submitted as `10000`, not `10000000000000000`.
- **Native Tokens**: Values follow the chain's native decimals.
- **Fee-related fields**: Certain fee values use field-specific scales, either explicitly defined by the field or determined by `FeeMode`. For example, platform fee rates and amount-based fee rates use `1e8`, whereas fixed fees use `1e6`.
- If a field defines its own `Scale` or explicitly specifies a different scale, follow that field-specific rule instead of the default one.

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
| sessionType  | uint8   | Y        |       | Trading Session: 0-DEFAULT, 1-PRE_MARKET, 2-AFTER_HOURS, 3-OVERNIGHT, 4-PRE_MARKET_AND_AFTER_HOURS                                                           |
| paymentToken | address | Y        |       | Payment token (stablecoin) used for order collateral and settlement                  |
| validDate    | uint32  | Y        |       | Valid days when TIF is GTD, range: [1,30]; otherwise set to 0                        |
| networkFee   | uint64  | Y        | 6     | Set to 0 when paying with native tokens. Cannot be used together with native tokens. |
| amount       | uint64  | Y        | 6     | Set to 0 (Not used for size-based market orders)                         |
| price        | uint48  | Y        | 6     | For MARKET orders: Worst acceptable price (Slippage Protection). Set to `Current Price * (1 + slippage)` for BUY, or `Current Price * (1 - slippage)` for SELL                       |
| size         | uint48  | Y        | 6     | Order size. Must be a whole number of shares                                                     |
| clientAddress| address | Y        |       | Client identifier expressed as an address in EVM address format. |

**Note:**

- The network fee paid in native token needs to have `value` set in the transaction.
- Network fee payment in stablecoins is not yet supported; `networkFee` should be set to `0`.
- `sessionType` accepts `DEFAULT`, `PRE_MARKET`, `AFTER_HOURS`, `OVERNIGHT`, and `PRE_MARKET_AND_AFTER_HOURS`. Non-`DEFAULT` session types can only be used with `LIMIT` orders.
- When the caller is also the client, `clientAddress` can be set to `0x0000000000000000000000000000000000000000`. For orders submitted by a B-side integrator contract, `account` is the caller contract address and `clientAddress` identifies the actual client. If the client does not use a self-custody wallet address, the B-side integrator must map the client's internal UID to an address in EVM address format and pass that mapped address as `clientAddress`.
- `clientAddress` is case-insensitive, and EIP-55 checksum validation is not required.
- Italicized values are not currently supported.

**Example:**

**Example: Caller Is Also the Client - Limit Order**

```cgo
const limitOrder: ITrading.NewOrder = {
    stockId: 1,
    tradeType: TradeType.LIMIT,
    side: Side.SELL,
    tif: Tif.DAY,
    sessionType: SessionType.DEFAULT, // Use PRE_MARKET, AFTER_HOURS, OVERNIGHT, or PRE_MARKET_AND_AFTER_HOURS for non-DEFAULT LIMIT orders
    paymentToken: "0x....",           // USDT address
    validDate: 0n,
    networkFee: 0n,
    amount: 0n,
    price: parseUnits("15", 6),
    size: parseUnits("5", 6),
    clientAddress: "0x0000000000000000000000000000000000000000",
};

ITrading.placeOrder(limitOrder, {value: 33000000000000})
```

**Example: Caller Is Also the Client - Market Order**

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
    clientAddress: "0x0000000000000000000000000000000000000000",
};

ITrading.placeOrder(marketOrder, {value: 33000000000000})
```

**Example: Order Submitted by a B-side Integrator Contract - Limit Order**

This example assumes the order is submitted by a B-side integrator contract. On-chain, `account` will be the caller contract address, and `clientAddress` will identify the actual client.

```cgo
const bSideLimitOrder: ITrading.NewOrder = {
    stockId: 1,
    tradeType: TradeType.LIMIT,
    side: Side.BUY,
    tif: Tif.DAY,
    sessionType: SessionType.DEFAULT,
    paymentToken: "0x....",           // USDT address
    validDate: 0n,
    networkFee: 0n,
    amount: 0n,
    price: parseUnits("15", 6),
    size: parseUnits("5", 6),
    clientAddress: "0x1234567890abcdef1234567890abcdef12345678", // Client UID mapped to an address in EVM address format
};

ITrading.placeOrder(bSideLimitOrder, {value: 33000000000000})
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
| chargedPlatformFee       | 	uint40	  | 6	     | Platform fees already charged            |
| chargedCommission | 	uint40	  | 6      | 	Broker commission already charged       |
| lockedPlatformFeeRate | uint32 | 8      |  Platform fee rate locked at order placement  |

**Example:**

```cgo
ITrading.getOrder(123456784567);
// Returns
{
    "account": "0xabcd...",
    "stockId": 1,
    "amount": 10000000,              // 10.000000
    "paymentToken": "0x1234...",
    "size": 1000000,                 // 1.000000
    "price": 10000000,               // 10.000000
    "tradeType": 0,
    "side": 0,
    "state": 0,
    "filledAmount": 900000,          // 0.900000
    "filledSize": 1000000,           // 1.000000
    "chargedPlatformFee": 6780,      // 0.006780
    "chargedCommission": 360000,     // 0.36
    "lockedPlatformFeeRate": 200000, // 0.002
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
| networkFeeInNative     | 	uint128	 | Y	       | By chain | 	Minimum network fee when paying with native tokens                                                           |
| networkFeeInStable	    | uint32    | 	Y       | 	6       | 	Minimum network fee when paying with stablecoins                                                             |
| minAmountPerOrder      | 	uint32   | 	Y	      | 6	       | Minimum order amount                                                                                          |
| maxAmountPerOrder      | 	uint64   | 	Y	      | 6	       | Maximum order amount                                                                                          |
| priceTTL               | 	uint32	  | Y	       |          | 	Oracle price validity period (seconds)                                                                       |
| limitPriceTolerate     | 	uint32	  | Y	       | 6	       | Maximum allowable deviation percentage between order price and oracle price for limit orders                  |
| marketPriceTolerate    | 	uint32	  | Y	       | 6	       | Maximum allowable deviation percentage between settlement price and oracle price for market orders            |
| maxReferralRate        | 	uint16   | 	Y       |  4        | Maximum referral rate                                                                                         |
| maxReferralVipLevel    | 	uint8    | 	Y       |          | Maximum referred-client VIP level for referral rebate; referral rebate applies only when the referred client's VIP level does not exceed this level                                                         |

**Example:**

```cgo
IMarket.getMarketConfig();
// Returns
{
    "actions": 3,                           // Both buying and selling are allowed
    "networkFeeInNative": 2500000000000,    // 0.0000025 (assuming 18 decimals)
    "networkFeeInStable": 2000,             // 0.002000
    "minAmountPerOrder": 20000000,          // 20.000000
    "maxAmountPerOrder": 1000000000,        // 1000.000000
    "priceTTL": 60,                         // 60 seconds
    "limitPriceTolerate": 10000,            // 0.01 (1%)
    "marketPriceTolerate": 50000,           // 0.05 (5%)
    "maxReferralRate": 3000,                // 0.3 (30%)
    "maxReferralVipLevel": 5              
}
```

### 2.2.2 Get Stock Config

`IMarket.getStockConfig(uint32 stockId) external returns (IMarket.StockConfig)`

**IMarket.StockConfig**

| Parameter	 | Type	    | Required | 	Scale	 | Description                                                                                                   |
|------------|----------|----------|---------|---------------------------------------------------------------------------------------------------------------| 
| token	     | address	 | Y	       |         | 	Contract address of the RWA asset                                                                            |
| actions    | 	uint64	 | Y	       |         | 	Allowed actions, represented as bits from right to left (bit set = allowed). refer to [Actions](#32-actions) |
| feeRate    | 	uint32	 | Y        | 	8	     | Per-stock platform fee rate override. A value of 0 means no override, and the global platform fee rate applies.                                                            |

**Example:**

```cgo
IMarket.getStockConfig(1);
// Returns
{
    "token": "0x...",
    "actions": 3,
    "feeRate": 400000      // 0.004
}
```

### 2.2.3 Get Fee Config

`IMarket.getFeeConfig() external view returns (uint32 platformFeeRate, IMarket.FeeRule[] buyFeeRules, IMarket.FeeRule[] sellFeeRules)`

Returns the current fee configuration, including the global platform fee rate and the brokerage fee rules for buy and sell orders.

**Returns**

| Parameter        | Type               | Required | Scale | Description                                                                 |
|------------------|--------------------|----------|-------|-----------------------------------------------------------------------------|
| platformFeeRate  | uint32             | Y        | 8     | Global platform fee rate, calculated based on the trade amount      |
| buyFeeRules      | IMarket.FeeRule[]  | Y        |       | Brokerage fee rules applied to buy orders                                   |
| sellFeeRules     | IMarket.FeeRule[]  | Y        |       | Brokerage fee rules applied to sell orders                                  |

**IMarket.FeeRule**

| Parameter | Type   | Required | Scale | Description                                                                 |
|-----------|--------|----------|-------|-----------------------------------------------------------------------------|
| ruleId    | uint8  | Y        |       | Identifier of the brokerage fee item                                        |
| mode      | uint8  | Y        |       | Base fee calculation mode. Determines the meaning and precision of `value`; must not be `NONE`. See `IMarket.FeeMode` below |
| minMode   | uint8  | Y        |       | Calculation mode for the minimum fee bound. `NONE` means no minimum limit and determines the meaning and precision of `minValue` |
| maxMode   | uint8  | Y        |       | Calculation mode for the maximum fee bound. `NONE` means no maximum limit and determines the meaning and precision of `maxValue` |
| value     | uint32 | Y        |       | Base fee value. Its unit and scale are determined by `mode`: `FIXED_FEE` = fixed amount scaled by `1e6`; `AMOUNT_RATIO` = amount-based rate scaled by `1e8`; `PER_SHARE` = per-share rate scaled by `1e8` |
| minValue  | uint32 | Y        |       | Minimum fee bound value. Its unit and scale are determined by `minMode` |
| maxValue  | uint32 | Y        |       | Maximum fee bound value. Its unit and scale are determined by `maxMode` |

**IMarket.FeeMode**

| Value | Name         | Description                                        |
|-------|--------------|----------------------------------------------------|
| 0     | NONE         | No fee mode. Used only for `minMode` and `maxMode` to indicate no minimum/maximum limit |
| 1     | FIXED_FEE    | Fixed fee amount, scaled by `1e6`                  |
| 2     | AMOUNT_RATIO | Amount-based fee rate, scaled by `1e8`             |
| 3     | PER_SHARE    | Per-share fee rate, scaled by `1e8`                |

**Example:**

```cgo
IMarket.getFeeConfig();
// Returns
{
    "platformFeeRate": 400000,   // 0.004
    "buyFeeRules": [
        {
            "ruleId": 1,
            "mode": 3,             // FeeMode.PER_SHARE (value scaled by 1e8)
            "minMode": 1,          // FeeMode.FIXED_FEE (minValue scaled by 1e6)
            "maxMode": 2,          // FeeMode.AMOUNT_RATIO (maxValue scaled by 1e8)
            "value": 350000,       // Base fee: 0.0035 per share
            "minValue": 350000,    // Minimum fee: 0.35
            "maxValue": 1000000    // Maximum fee: 1% of trade amount
        }
    ],
    "sellFeeRules": [
        {
            "ruleId": 1,           // Same rule as buyFeeRules ruleId 1
            "mode": 3,
            "minMode": 1,
            "maxMode": 2,
            "value": 350000,
            "minValue": 350000,
            "maxValue": 1000000
        }, 
        {
          "ruleId": 2,
          "mode": 3,               // FeeMode.PER_SHARE (value scaled by 1e8)
          "minMode": 1,            // FeeMode.FIXED_FEE (minValue scaled by 1e6)
          "maxMode": 1,            // FeeMode.FIXED_FEE (maxValue scaled by 1e6)
          "value": 19500,          // Base fee: 0.000195 per share
          "minValue": 10000,       // Minimum fee: 0.01
          "maxValue": 9790000      // Maximum fee: 9.79
        }
    ]
}
```

Note: The same `ruleId` represents the same brokerage fee rule definition. If a rule applies to both buy and sell orders, it may appear in both `buyFeeRules` and `sellFeeRules` with the same `ruleId`.

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
| 2   | 4     | Claim Rebate   |

**Example:**
- 0: Neither buying nor selling is allowed.
- 1: Only buying is allowed.
- 2: Only selling is allowed.
- 3: Both buying and selling are allowed.
