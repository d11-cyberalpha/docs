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
- 2026年5月14日
  - 修改: 实时行情 汇总行情 聚合价格
  - 新增: 市场状态 

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
  "args": ["price.AAPLt", "price.COINt", "realtime.AAPLt", "summary"]
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
  "args": ["price.AAPLt", "price.COINt", "realtime.AAPLt", "summary"]
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
      "data": "price.AAPLt"
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
  "type": "price.AAPLt",
  "data": {
    "s":1,          // 股票编号
    "p":258.04005,  // 最新价格
    "t":1759953620  // 时间戳(秒)
  }
}
```

## 2.2 实时行情

**主题:** `realtime.{stock}`

**推送频率:** 实时

**Message:**

```json5
{
  "type": "realtime.AAPLt",
  "data": {
    "s":1,          // 股票编号    
    "p":258.04005,  // 最新价格
    "o":256.46838,  // 24小时开始价格
    "l":256.15202,  // 最低价
    "h":258.48045,  // 最高价
    "T":1768176000  // 时间戳(秒)
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
       "s":1,           // 股票编号       
       "S":"AAPLt",     // 股票Symbol
       "p":258.04005,   // 最新价格
       "o":256.46838,   // 24小时开始价格
       "l":256.15202,   // 最低价
       "h":258.48045,   // 最高价
       "T":1759881600   // 时间戳(秒)
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
## 2.4 聚合价格

**主题:** `aggregate`

**推送频率:** 1次/秒

**Message:**

```json5
{
  "type": "aggregate",
  "data": {
    "timestamp":1759953620,  // 时间戳(秒)
    "items":[
      {
        "s":1,              // 股票编号       
        "S":"AAPLt",        // 股票Symbol
        "p":258.04005,      // 最新价格
        "o":258.04005       // 24小时开始价格
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
  "type": "candle.AAPLt_1h",
  "data": {
    "t": 1725811200,    // K线周期起始时间(Unix时间戳: 秒)
    "o": 65633.66,      // 开盘价
    "c": 65633.66,      // 收盘价
    "h": 66123.45,      // 最高价
    "l": 65000.00,      // 最低价
    "v": 0.0,           // 成交量
    "T": 0.0,           // 成交额
    "E": 1725844149000  // Event time(Unix时间戳: 毫秒)
  } 
}
```
## 2.6 市场状态

**主题:** `marketState`

**推动频率:** 1次/秒

**Message:**

```json5
{
  "type": "marketState",
  "data": {
    "m": "us",          // 市场 (Market): "us"-US, "hk"-HK
    "s": 0,             // 盘段状态 (SessionType) 
                        // 0-NOT_OPEN (未开盘)
                        // 1-US_PRE_MARKET (盘前)
                        // 2-TRADING (盘中)
                        // 3-US_AFTER_HOURS (盘后)
                        // 4-CLOSED (已收盘)
                        // 5-US_NIGHT_SESSION (夜盘)
    "d": 0,             // 交易日类型 (TradingDayType): 
                        // 0-DAY_TYPE_FULL (全天交易)
                        // 1-DAY_TYPE_HALF_MORNING (午前半天)
                        // 2-DAY_TYPE_HALF_AFTERNOON (午后半天)
    "L": 7,             // 限价单交易时段权限，如盘前时段：0111-夜盘-盘后-盘前-盘中
    "M": 0,             // 市价单交易时段权限，如盘前时段：0000-夜盘-盘后-盘前-盘中
    "t": 1725844149000  // 毫秒时间戳 (Timestamp)
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
  "type": "order.97.AAPLt",
  "data": {
    "hx": "0xabcxxx",   // 交易哈希
    "id": 12345678,     // 订单编号
    "si": 1,            // 股票编号
    "p": 65535.00,      // 委托价格
    "s": 0.12,          // 委托数量
    "S": "BUY",         // 买卖方向: BUY/SELL
    "y": "LIMIT",       // 订单类型: MARKET/LIMIT
    "x": "NEW",         // 订单状态
    "R": "FREE",        // 风险类型
    "N": 0.000123,      // 网络手续费(稳定币支付)
    "V": 0.01,          // 网络手续费(原生代币支付)
    "f": "GTD",         // 订单有效时间
    "d": 7,             // 有效期
    "st": "DEFAULT",    // 交易时段类型
    "T": "USDC",        // 支付代币
    "m": 234.23,        // 订单金额
    "t": 1725844149,    // 更新时间(Unix时间戳: 秒)
    "E": 1725844149000  // Event time(Unix时间戳: 毫秒)
  }
}
```

* Execute order

```json5
{
  "type": "order.97.AAPLt",
  "data": {
    "hx": "0xabcxxx",   // 交易哈希
    "id": 12345678,     // 订单编号
    "si": 1,            // 股票编号
    "p": 65535.00,      // 委托价格
    "s": 0.12,          // 委托数量
    "S": "BUY",         // 买卖方向: BUY/SELL
    "y": "LIMIT",       // 订单类型: MARKET/LIMIT
    "C": 0,             // 累计交易量
    "c": 0,             // 累计交易额
    "x": "FILLED",      // 订单状态: FILLED/PARTIALLY_FILLED
    "R": "Free",        // 风险类型
    "F": 0.000111,      // 手续费
    "T": "USDC",        // 支付代币
    "t": 1725844149,    // 更新时间(Unix时间戳: 秒)
    "E": 1725844149000  // Event time(Unix时间戳: 毫秒)
  }
}
```

* Cancelled/Expired order

```json5
{
  "type": "order.97.AAPLt",
  "data": {
    "hx": "0xabcdxxx",            // 交易哈希
    "id": 12345678,               // 订单编号
    "si": 1,                      // 股票编号
    "y": "LIMIT",                 // 订单类型: MARKET/LIMIT
    "x": "CANCELED",              // 订单状态
    "R": "FREE",                  // 风险类型
    "r": 1,                       // 原因
    "t": 1725844149,              // 更新时间(Unix时间戳: 秒)
    "E": 1725844149000            // Event time(Unix时间戳: 毫秒)
  }
}
```

* PendingCancel order

```json5
{
  "type": "order.97.AAPLt",
  "data": {
    "hx": "0xabcde",              // 交易哈希
    "id": 12345678,               // 订单编号
    "si": 1,                      // 股票编号
    "x": "PENDING_CANCEL",        // 订单状态
    "t": 1725844149,              // 更新时间(Unix时间戳: 秒)
    "E": 1725844149000            // Event time(Unix时间戳: 毫秒)
  }
}
```