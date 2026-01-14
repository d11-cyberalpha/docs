<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Web Socket 接口](#web-socket-接口)
  - [更新日志](#更新日志)
- [1. 基础信息](#1-基础信息)
  - [1.1 接入点](#11-接入点)
  - [1.2 流控](#12-流控)
  - [1.3 消息交互](#13-消息交互)
    - [1.3.1 消息格式](#131-消息格式)
    - [1.3.2 订阅](#132-订阅)
    - [1.3.3 取消订阅](#133-取消订阅)
    - [1.3.4 Ping/Pong](#134-pingpong)
    - [1.3.5 认证](#135-认证)
    - [1.3.6 响应码与响应消息](#136-响应码与响应消息)
  - [1.4 支持的链](#14-支持的链)
- [2 公共频道](#2-公共频道)
  - [2.1 实时价格](#21-实时价格)
  - [2.2 实时行情](#22-实时行情)
  - [2.3 汇总行情](#23-汇总行情)
  - [2.4 聚合价格](#24-聚合价格)
  - [2.5 K线](#25-K线)
- [3 私有频道](#3-私有频道)
  - [3.1 订单](#31-订单)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Web Socket 接口

## 更新日志

- 2026年1月11日
  - 首次发布

# 1. 基础信息

- WebSocket 服务器会每隔 20 秒发送一次 `ping` 事件。
  - 如果 WebSocket 服务器在 60 秒内未从该连接收到返回的 `pong` 事件，则会断开此连接。
  - 当收到 `ping` 事件时，必须立即返回一个 `pong` 事件，且需携带该 `ping` 事件的负载数据。

## 1.1 接入点

**测试环境**
  
  `wss://stream-test.cyberalpha.cc/ws`

**生产环境**
  
  `wss://stream.tiko.cc/ws`

## 1.2 流控

* **同一 IP 每 60 秒内的连接数上限为 50 个。**.
* 任何超出连接限制阈值的请求连接，都将被服务端主动断开。
* 频繁被断开连接的 IP，有可能会被禁用。

## 1.3 消息交互

### 1.3.1 消息格式

- Request(客户端到服务器端)

| 字段        | 值/类型                        | 说明   |
|-----------|-----------------------------|------|
| request   | `sub`/`unsub`/`pong`/`auth` |      |
| requestId | long                        | 可选   |
| args      | object                      |      |

- Response(server's response)

| 字段        | 值/类型                        | 说明                   |
|-----------|-----------------------------|----------------------|
| type      | `sub`/`unsub`/`ping`/`auth` |                      |
| requestId | long                        |                      |
| data      | object                      |                      |
| - code    | integer                     | 响应码                  |
| - message | string                      | 响应消息, **如果为空,可能不返回** |
| - data    | object                      | Data                 |

- Push(服务端推送给客户端)

| 字段   | 值/类型           | 说明          |
|------|----------------|-------------|
| type | `{topic name}` | 主题名称        |
| data | object         | 不同主题的数据格式不同 |

### 1.3.2 订阅

**args**: 主题数组

- Request

```json5
{
  "request": "sub",
  "requestId": 123456,
  "args": ["price.AAPLc", "price.COINc", "realtime.AAPLc", "summary"]
}
```

- Response(成功)

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

- Response(失败)

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

### 1.3.3 取消订阅

**args**: 主题数组

- Request

```json5
{
  "request": "unsub",
  "requestId": 123456,
  "args": ["price.AAPLc", "price.COINc", "realtime.AAPLc", "summary"]
}
```

- Response(成功)

  *当同时订阅多个主题时，每个主题的响应将分别单独推送*

```json5
  {
    "type": "unsub",
    "requestId": 123456,
    "data": {
      "code": 9200,
      "data": "price.AAPLc"
    }
  }
```

- Response(失败)

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

`ping` 消息由服务端发送，客户端接收到该消息后，需携带 `ping` 消息的数据向服务端回复 `pong` 消息。

**若服务端连续三次未收到 `pong` 消息，则会关闭连接。**

- Ping

```json5
{
  "type": "ping",
  "data": 1725844149000 // Unix timestamp(ms)
}
```

- Pong

```json5
{
  "request": "pong",
  "args": 1725844149000 // the data of the ping message
}
```

### 1.3.5 认证

**args**: 认证参数

- Request

```json5
{
  "request": "auth",
  "requestId": 123456,
  "args":"Bearer xxxx"
}
```

- Response(成功)

```json5
{
  "type": "auth",
  "requestId": 123456,
  "data":{
     "code":9200
   }
}
```

- Response(失败)

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

### 1.3.6 响应码与响应消息

- 9200: **No message**

  *服务器成功处理了客户端的请求。*

- 9901: **Illegal parameter**

  *参数错误，包括：格式、范围、数值等。*

- 9401: **Unauthorized**

  *用户认证失败，包括签名错误、过期等情况。*

- 9403: **Forbidden**

  *无权限操作，例如未经授权访问。*

- 9429: **Too many requests**

  *访问频率超过限定频率。*

- 9500: **Internal server error**

  *服务器端未知错误。*

## 1.4 支持的链

**测试环境**

| 链名称     | 链 ID |
|---------|------|
| BSC     | 97   |
| X Layer | 195  |

**生产环境**

| 链名称     | 链 ID |
|---------|------|
| BSC     | 56   |
| X Layer | 196  |

# 2 公共频道

## 2.1 实时价格

**主题:** `price.{stock}`

**推送频率:** 实时

**Message:**

```json5
{
  "type": "price.AAPLc",
  "data": {
    "s":1,          // Stock ID
    "p":258.04005,  // Last price
    "t":1759953620  // Timestamp(second)
  }
}
```

## 2.2 实时行情

**主题:** `realtime.{stock}`

**推送频率:** 实时

**Message:**

```json5
{
  "type": "realtime.AAPLc",
  "data": {
    "s":1,          // Stock ID    
    "p":258.04005,  // Last price
    "o":256.46838,  // Today's opening price
    "l":256.15202,  // Today's lowest price
    "h":258.48045,  // Today's highest price
    "c":258.04005,  // Today's closing price
    "pc":256.45015, // Previous closing price
    "T":1768176000  // Timestamp(second)
  }
}
```
## 2.3 汇总行情

**主题:** `summary`

**推送频率:** 1次/秒

**Message:**

```json5
{
  "type": "summary",
  "data": [
     {
       "s":1,           // Stock ID       
       "S":"AAPLc",     // Stock symbol
       "p":258.04005,   // Last price
       "o":256.46838,   // Opening price
       "l":256.15202,   // Lowest price
       "h":258.48045,   // Highest price
       "c":258.04005,   // Closing price
       "pc":256.45015,  // Previous closing price
       "T":1759881600   // Timestamp(second)
     },{
       "s":2,
       "S":"COINc",
       "p":387.2823,
       "o":377.03758,
       "l":375.95872,
       "h":390.3395,
       "c":387.2823,
       "pc":375.6325,
       "T":1759881600
     }
  ]
}
```
## 2.4 聚合价格

**主题:** `aggregate`

**推送频率:** 1次/秒

**Message:**

```json5
{
  "type": "aggregate",
  "data": {
    "timestamp":1759953620,  // Timestamp(second)
    "items":[
      {
        "s":1,              // Stock ID       
        "S":"AAPLc",        // Stock Symbol
        "p":258.04005,      // Last price
        "pc":258.04005      // Previous closing price
      },
      {
        "s":2,
        "S":"COINc",
        "p":387.2823,    
        "pc":258.04005,
      }
    ]
  } 
}
```
## 2.5 K线

**主题:** `candle.{stock}_{interval}`

**Interval:** `1m`/`5m`/`15m`/`30m`/`1h`/`4h`/`1d`/`1w`/`1M`
- m: minutes
- h: hours
- d: days
- w: weeks
- M: months

**推动频率:** 实时

**Message:**

```json5
{
  "type": "candle.AAPLc_1h",
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

# 3 私有频道

该频道的数据需要先经过认证才能订阅。

## 3.1 订单

**主题:** `order.{chainId}.{stock}`

*如果订阅该链所有股票的订单变更，stock传`*`。*

**推送频率:** 实时

**Message:**

- New order

```json5
{
  "type": "order.97.AAPLc",
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
  "type": "order.97.AAPLc",
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
  "type": "order.97.AAPLc",
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
  "type": "order.97.AAPLc",
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