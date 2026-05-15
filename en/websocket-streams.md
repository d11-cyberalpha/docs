<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Web Socket Interface](#web-socket-interface)
  - [Changelog](#changelog)
- [1. General information](#1-general-information)
  - [1.1 Endpoints](#11-endpoints)
  - [1.2 Rate limits](#12-rate-limits)
  - [1.3 Message interactions](#13-message-interactions)
    - [1.3.1 Message format](#131-message-format)
    - [1.3.2 Subscribe](#132-subscribe)
    - [1.3.3 Unsubscribe](#133-unsubscribe)
    - [1.3.4 Ping/Pong](#134-pingpong)
    - [1.3.5 Authentication](#135-authentication)
    - [1.3.6 Response codes and messages](#136-response-codes-and-messages)
  - [1.4 Supported chains](#14-supported-chains)
- [2 Public stream information](#2-public-stream-information)
  - [2.1 Real-time price](#21-real-time-price)
  - [2.2 Real-time quotes](#22-real-time-quotes)
  - [2.3 Market summary](#23-market-summary)
  - [2.4 Aggregated price](#24-aggregated-price)
  - [2.5 Candle](#25-candle)
- [3 Private stream information](#3-private-stream-information)
  - [3.1 Order information](#31-order-information)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Web Socket Interface

## Changelog

- May 14, 2026
  - Modified: Realtime Summary Aggregated
  - Added: Market Status
- January 11, 2026
  - Initial release

# 1. General information

- Websocket server will send a `ping` event every 20 seconds.
  - If the websocket server does not receive a `pong` event back from the connection within a 60 seconds period, the connection will be disconnected.
  - When you receive a ping, you must send a pong with a copy of ping's payload as soon as possible.

## 1.1 Endpoints

**Beta Env**
  
  `wss://stream-test.cyberalpha.cc/ws`

**Prod Env**
  
  `wss://stream.tiko.cc/ws`

## 1.2 Rate limits

* There is a limit of **50 connections per attempt every 60 seconds per IP**.
* Connections exceeding the limit will be disconnected.
* Repeatedly disconnected IPs may be disabled.

## 1.3 Message interactions

### 1.3.1 Message format

* Request(client to server)

| field     | value                       | note     |
|-----------|-----------------------------|----------|
| request   | `sub`/`unsub`/`pong`/`auth` |          |
| requestId | long                        | Optional |
| args      | object                      |          |

- Response(server's response)

| field     | value                       | note                                              |
|-----------|-----------------------------|---------------------------------------------------|
| type      | `sub`/`unsub`/`ping`/`auth` |                                                   |
| requestId | long                        |                                                   |
| data      | object                      |                                                   |
| - code    | integer                     | Response code                                     |
| - message | string                      | Response message, **May not be returned if null** |
| - data    | object                      | Data                                              |

- Push(server push to the client)

| field | value          | note                               |
|-------|----------------|------------------------------------|
| type  | `{topic name}` | The topic name for the pushed data |
| data  | object         | The data format varies by topic    |

### 1.3.2 Subscribe

**args**: Topic array

- Request

```json5
{
  "request": "sub",
  "requestId": 123456,
  "args": ["price.AAPLt", "price.COINt", "realtime.AAPLt", "summary"]
}
```

- Response(Successful)

```json5
{
  "type": "sub",
  "requestId": 123456,
  "data": {
    "code": 9200,
    "data": {}
  }
}
```

* Response(Failed)

```json5
{
  "type": "sub",
  "requestId": 123456,
  "data": {
    "code": 9901,
    "message":"Illegal Parameter"
  }
}
```

### 1.3.3 Unsubscribe

**args**: Topic array
- Request

```json5
{
  "request": "unsub",
  "requestId": 123456,
  "args": ["price.AAPLt", "price.COINt", "realtime.AAPLt", "summary"]
}
```

* Response(Successful)

  When multiple streams are subscribed at once, the responses for each stream will be sent separately.

```json5
  {
    "type": "unsub",
    "requestId": 123456,
    "data": {
      "code": 9200,
      "data": "price.AAPLt"
    }
  }
```

* Response(Failure)

```json5
  {
    "type": "unsub",
    "requestId": 123456,
    "data": {
      "code": 9901,
      "message":"Illegal Parameter"
    }
  }
```

### 1.3.4 Ping/Pong

Ping messages are sent by the server, and when the client receives them, it needs to reply to the server with a pong message using the data of the ping.

**If the server fails to receive a pong message for three consecutive times, it will close the connection.**

- Ping

```json5
{
  "type": "ping",
  "data": 1725844149000 // Unix timestamp(ms)
}
```

* Pong

```json5
{
  "request": "pong",
  "args": 1725844149000 // the data of the ping message
}
```

### 1.3.5 Authentication

**args**: Auth parameters
- Request

```json5
{
  "request": "auth",
  "requestId": 123456,
  "args":"Bearer xxxx"
}
```

* Response(Successful)

```json5
{
  "type": "auth",
  "requestId": 123456,
  "data":{
     "code":9200
   }
}
```

* Response(Failed)

```json5
{
  "type": "auth",
  "requestId": 123456,
  "data":{
    "code":9401,
    "message":"signature invalid"
  }
}
```

### 1.3.6 Response codes and messages

- 9200: **No message**

  *The server successfully handled the client's request.*
- 9901: **Illegal parameter**

  *Parameter errors, including: format, range, value, etc.*
- 9401: **Unauthorized**

  *Authentication failure of the user, including signature error, expiration, etc.*
- 9403: **Forbidden**

  *Unprivileged operations, e.g., unauthorized access to private stream.*
- 9429: **Too many requests**

  Rate limit exceeded.
- 9500: **Internal server error**

  *Any unknown error not caused by the client.*

## 1.4 Supported chains

**Beta Env**

| Chain Name | Chain ID |
|------------|----------|
| BSC        | 97       |
| X Layer    | 195      |

**Prod Env**

| Chain Name | Chain ID |
|------------|----------|
| BSC        | 56       |
| X Layer    | 196      |

# 2 Public stream information

## 2.1 Real-time price

**Topic:** `price.{stock}`

**Push Frequency:** Real-time

**Message:**

```json5
{
  "type": "price.AAPLt",
  "data": {
    "s":1,          // Stock ID
    "p":258.04005,  // Last price
    "t":1759953620  // Timestamp(second)
  }
}
```

## 2.2 Real-time quotes

**Topic:** `realtime.{stock}`

**Push Frequency:** Real-time

**Message:**

```json5
{
  "type": "realtime.AAPLt",
  "data": {
    "s":1,          // Stock ID    
    "p":258.04005,  // Last price
    "o":256.46838,  // 24-hour opening price
    "l":256.15202,  // Today's lowest price
    "h":258.48045,  // Today's highest price
    "T":1768176000  // Timestamp(second)
  }
}
```
## 2.3 Market summary

**Topic:** `summary`

**Push Frequency:** 1/second

**Message:**

```json5
{
  "type": "summary",
  "data": [
     {
       "s":1,           // Stock ID       
       "S":"AAPLt",     // Stock symbol
       "p":258.04005,   // Last price
       "o":256.46838,   // 24-hour opening price
       "l":256.15202,   // Lowest price
       "h":258.48045,   // Highest price
       "T":1759881600   // Timestamp(second)
     },{
       "s":2,
       "S":"COINt",
       "p":387.2823,
       "o":377.03758,
       "l":375.95872,
       "h":390.3395,
       "T":1759881600
     }
  ]
}
```
## 2.4 Aggregated price

**Topic:** `aggregate`

**Push Frequency:** 1/second

**Message:**

```json5
{
  "type": "aggregate",
  "data": {
    "timestamp":1759953620,  // Timestamp(second)
    "items":[
      {
        "s":1,              // Stock ID       
        "S":"AAPLt",        // Stock Symbol
        "p":258.04005,      // Last price
        "o":258.04005       // 24-hour opening price
      },
      {
        "s":2,
        "S":"COINt",
        "p":387.2823,    
        "o":258.04005,
      }
    ]
  } 
}
```
## 2.5 Candle

**Topic:** `candle.{stock}_{interval}`

**Interval:** `1m`/`5m`/`15m`/`30m`/`1h`/`4h`/`1d`/`1w`/`1M`
- m: minutes
- h: hours
- d: days
- w: weeks
- M: months

**Push Frequency:** Real-time

**Message:**

```json5
{
  "type": "candle.AAPLt_1h",
  "data": {
    "t": 1725811200,    // Candle start time(Unix timestamp: s)
    "o": 65633.66,      // Open price
    "c": 65633.66,      // Close price
    "h": 66123.45,      // High price
    "l": 65000.00,      // Low price
    "v": 0.0,           // Volume
    "T": 0.0,           // Turnover
    "E": 1725844149000  // Event time(Unix timestamp: ms)
  } 
}
```
## 2.6 Market Status

**Topic:** `marketState`

**Push Frequency:** 1次/秒

**Message:**

```json5
{
  "type": "marketState",
  "data": {
    "m": "us",          // Market: "us"-US, "hk"-HK
    "s": 0,             // SessionType:
                        // 0-NOT_OPEN 
                        // 1-US_PRE_MARKET
                        // 2-TRADING
                        // 3-US_AFTER_HOURS
                        // 4-CLOSED 
                        // 5-US_NIGHT_SESSION 
    "d": 0,             // TradingDayType: 
                        // 0-DAY_TYPE_FULL
                        // 1-DAY_TYPE_HALF_MORNING
                        // 2-DAY_TYPE_HALF_AFTERNOON
    "L": 7,             // Limit Order Trading Session Permissions: 
                        // e.g. Pre-market Session: 0111 - Night Session - After-hours - Pre-market - Regular Trading Hours
    "M": 0,             // Market Order Trading Session Permissions:
                        // e.g. Pre-market Session: 0000 - Night Session - After-hours Session - Pre-market Session - Regular Trading Session
    "t": 1725844149000  // Millisecond Timestamp
  } 
}
```

# 3 Private stream information

Access to this channel's data requires authentication before subscription.

## 3.1 Order information

**Topic:** `order.{chainId}.{stock}`

*To subscribe to messages for all stocks in the chain, set stock parameter to `*`.*

**Push Frequency:** Real-time

**Message:**

- New order

```json5
{
  "type": "order.97.AAPLt",
  "data": {
    "hx": "0xabcxxx",   // tx_hash
    "id": 12345678,     // Order id
    "si": 1,            // Stock id
    "p": 65535.00,      // Order price
    "s": 0.12,          // Order size
    "S": "BUY",         // Order side: BUY/SELL
    "y": "LIMIT",       // Trade type: MARKET/LIMIT
    "x": "NEW",         // Order status
    "R": "FREE",        // Risk type
    "N": 0.000123,      // Network fee(Stable Token)
    "V": 0.01,          // Network fee(Native Token)
    "f": "GTD",         // Time in force
    "d": 7,             // Valid date
    "st": "DEFAULT",    // SessionType
    "T": "USDC",        // Token of payment
    "m": 234.23,        // Order amount
    "t": 1725844149,    // Update time(Unix timestamp: s)
    "E": 1725844149000  // Event time(Unix timestamp: ms)
  }
}
```

* Execute order

```json5
{
  "type": "order.97.AAPLt",
  "data": {
    "hx": "0xabcxxx",   // tx_hash
    "id": 12345678,     // Order id
    "si": 1,            // Stock id
    "p": 65535.00,      // Order price
    "s": 0.12,          // Order size
    "S": "BUY",         // Order side: BUY/SELL
    "y": "LIMIT",       // Trade type: MARKET/LIMIT
    "C": 0,             // Cumulative trade size
    "c": 0,             // Cumulative trade amount
    "x": "FILLED",      // Order status: FILLED/PARTIALLY_FILLED
    "R": "Free",        // Risk type
    "F": 0.000111,      // fee
    "T": "USDC",        // Token of payment
    "t": 1725844149,    // Update time(Unix timestamp: s)
    "E": 1725844149000  // Event time(Unix timestamp: ms)
  }
}
```

* Cancelled/Expired order

```json5
{
  "type": "order.97.AAPLt",
  "data": {
    "hx": "0xabcdxxx",            // tx_hash
    "id": 12345678,               // Order id
    "si": 1,                      // Stock id
    "y": "LIMIT",                 // Order type: MARKET/LIMIT
    "x": "CANCELED",              // Order status
    "R": "FREE",                  // Risk type
    "r": 1,                       // Reason
    "t": 1725844149,              // Update time(Unix timestamp: s)
    "E": 1725844149000            // Event time(Unix timestamp: ms)
  }
}
```

* PendingCancel order

```json5
{
  "type": "order.97.AAPLt",
  "data": {
    "hx": "0xabcde",              // tx_hash
    "id": 12345678,               // Order id
    "si": 1,                      // Stock id
    "x": "PENDING_CANCEL",        // Order status
    "t": 1725844149,              // Update time(Unix timestamp: s)
    "E": 1725844149000            // Event time(Unix timestamp: ms)
  }
}
```