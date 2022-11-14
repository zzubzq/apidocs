---
title: Pexpay API Documentation
language_tabs: # must be one of https://git.io/vQNgJ
  #- shell
  #- javascript
  #- json

toc_footers:
  - <a href='https://www.pexpay.co/cn/'>Pexpay Exchange</a>

includes:

search: true

---


# 更新日志

<font size=4>**2018-01-14**</font>

* GET/api/v1/aggTrades权重更改为2
* GET/api/v1/klines权重更改为2
* GET/api/v3/订单权重更改为2
* GET/api/v3/ allOrders权重更改为20
* GET/api/v3/帐户权重更改为20
* GET/api/v3/ myTrades权重更改为20
* GET/api/v3/ historicalTrades权重更改为20


# 介绍

## API Key 设置

* 很多接口需要API Key才可以访问. 请参考[这个页面](https://www.pexpayzh.top/zh-CN/support/faq/360002502072)来设置API Key.
* 设置API Key的同时，为了安全，建议设置IP访问白名单.
* **永远不要把你的API key/secret告诉给任何人**

<aside class="warning">
如果不小心泄露了API key，请立刻删除此Key, 并可以另外生产新的Key.
</aside>


## 联系我们

* [请在这里提交您的请求或问题](https://support.pexpay.com/hc/en-us/requests/new?ticket_form_id=11868568981785)


# 基本信息
## API 基本信息

* 接口可能需要用户的 API Key，如何创建API-KEY请参考[这里](https://www.pexpay.com/cn/support/articles/360002502072)
* 本篇列出接口的baseurl: **https://api.pexpay.com**
* 所有接口的响应都是 JSON 格式。
* 所有时间、时间戳均为UNIX时间，单位为**毫秒**。

### HTTP 返回代码
* HTTP `4XX` 错误码用于指示错误的请求内容、行为、格式。问题在于请求者。
* HTTP `403` 错误码表示违反WAF限制(Web应用程序防火墙)。
* HTTP `429` 错误码表示警告访问频次超限，即将被封IP。
* HTTP `418` 表示收到429后继续访问，于是被封了。
* HTTP `5XX` 错误码用于指示Pexpay服务侧的问题。    


### 接口错误代码

* 每个接口都有可能抛出异常; 

> SAPI 的错误代码返回形式如下:

```javascript
{
  "code": ${error code},
  "msg": "error message."
}
```

* 具体的错误码及其解释在 [错误代码](#cf68bca02a).

### 接口的基本信息

* `GET` 方法的接口, 参数必须在 `query string`中发送。
* `POST`, `PUT`, 和 `DELETE` 方法的接口,参数可以在内容形式为`application/x-www-form-urlencoded`的 `query string` 中发送，也可以在 `request body` 中发送。 如果你喜欢，也可以混合这两种方式发送参数。
* 对参数的顺序不做要求。
* 但如果同一个参数名在query string和request body中都有，query string中的会被优先采用。
  
---
## 访问限制
### 访问限制基本信息
* 以下 是`intervalLetter` 作为头部值:
    * SECOND => S
    * MINUTE => M
    * HOUR => H
    * DAY => D
   
* 在 `/api/v3/exchangeInfo` `rateLimits` 数组中包含与交易的有关RAW_REQUESTS，REQUEST_WEIGHT和ORDERS速率限制相关的对象。这些在 `限制种类 (rateLimitType)` 下的 `枚举定义` 部分中进一步定义。

* 违反任何一个速率限制时，将返回429。

### IP 访问限制
* 每个请求将包含一个`X-MBX-USED-WEIGHT-(intervalNum)(intervalLetter)`的头，其中包含当前IP所有请求的已使用权重。
* 每一个接口均有一个相应的权重(weight)，有的接口根据参数不同可能拥有不同的权重。越消耗资源的接口权重就会越大。
* 收到429时，您有责任停止发送请求，不得滥用API。
* **收到429后仍然继续违反访问限制，会被封禁IP，并收到418错误码**
* 频繁违反限制，封禁时间会逐渐延长，**从最短2分钟到最长3天**。
* `Retry-After`的头会与带有418或429的响应发送，并且会给出**以秒为单位**的等待时长(如果是429)以防止禁令，或者如果是418，直到禁令结束。
* **访问限制是基于IP的，而不是API Key**


###下单频率限制
* 每个成功的下单回报将包含一个`X-MBX-ORDER-COUNT-(intervalNum)(intervalLetter)`的头，其中包含当前账户已用的下单限制数量。
* 当下单数超过限制时，会收到带有429但不含`Retry-After`头的响应。请检查 `GET api/v3/exchangeInfo` 的下单频率限制 (rateLimitType = ORDERS) 并等待封禁时间结束。
* 被拒绝或不成功的下单并不保证回报中包含以上头内容。
* **下单频率限制是基于每个账户计数的。**
* 用户可以通过接口 `GET api/v3/rateLimit/order` 来查询当前的下单量.


### /sapi/ 接口限频说明

*  `/sapi/*`的接口相关：
	* 按IP和按UID(account)两种模式分别统计, 两者互相独立。
	*  以`/sapi/*`开头的接口采用**单接口限频模式**。按IP统计的权重单接口权重总额为每分钟12000；按照UID统计的单接口权重总额是每分钟180000。
	*  每个接口会标明是按照IP或者按照UID统计, 以及相应请求一次的权重值。
	*  按照IP统计的接口, 请求返回头里面会包含`X-SAPI-USED-IP-WEIGHT-1M=<value>`, 包含当前IP所有请求已使用权重。
	*  按照UID统计的接口, 请求返回头里面会包含`X-SAPI-USED-UID-WEIGHT-1M=<value>`, 包含当前账户所有已用的UID权重。

---
## SIGNED (TRADE、USER_DATA AND MARGIN) Endpoint security
* 调用`SIGNED` 接口时，除了接口本身所需的参数外，还需要在`query string` 或 `request body`中传递 `signature`, 即签名参数。
* 签名使用`HMAC SHA256`算法. API-KEY所对应的API-Secret作为 `HMAC SHA256` 的密钥，其他所有参数作为`HMAC SHA256`的操作对象，得到的输出即为签名。
* `签名` **大小写不敏感**.
* "totalParams"定义为与"request body"串联的"query string"。

### 时间同步安全
* 签名接口均需要传递 `timestamp`参数，其值应当是请求发送时刻的unix时间戳(毫秒)。
* 服务器收到请求时会判断请求中的时间戳，如果是5000毫秒之前发出的，则请求会被认为无效。这个时间空窗值可以通过发送可选参数 `recvWindow`来定义。

> 逻辑伪代码如下:

```javascript
  if (timestamp < (serverTime + 1000) && (serverTime - timestamp) <= recvWindow)
  {
    // process request
  } 
  else 
  {
    // reject request
  }
```

**关于交易时效性** 互联网状况并不完全稳定可靠,因此你的程序本地到传奇服务器的时延会有抖动。这是我们设置`recvWindow`的目的所在，如果你从事高频交易，对交易时效性有较高的要求，可以灵活设置`recvWindow`以达到你的要求。

<aside class="notice">
推荐使用5秒以下的 recvWindow! 最多不能超过 60秒!
</aside>

# C2C 接口


## 获取 C2C 交易历史记录 (USER_DATA)

> **响应:**

```javascript
{
   "code": "000000",
   "message": "success",
   "data": [
   {
      "orderNumber":"20219644646554779648",
      "advNo": "11218246497340923904",
      "tradeType": "SELL",  
      "asset": "BUSD", 
      "fiat": "CNY",
      "fiatSymbol": "￥",
      "amount": "5000.00000000",  // Quantity (in Crypto)
      "totalPrice": "33400.00000000",
      "unitPrice": "6.68", // Unit Price (in Fiat)
      "orderStatus": "COMPLETED",  // PENDING, TRADING, BUYER_PAYED, DISTRIBUTING, COMPLETED, IN_APPEAL, CANCELLED, CANCELLED_BY_SYSTEM
      "createTime": 1619361369000,
      "commission": "0",  // Transaction Fee (in Crypto)
      "counterPartNickName": "阿涛❤***",
      "advertisementRole": "TAKER"        
     }
   ],
   "total": 1,
   "success": true
}
```

``
GET /sapi/v1/c2c/orderMatch/listUserOrderHistory (HMAC SHA256)
``


**权重(IP):**
1

**参数:**

| 名称           | 类型   | 是否必需 | 描述                 |
| -------------- | ------ | -------- | -------------------- |
| tradeType      | STRING | YES      | BUY, SEll            |
| startTimestamp | LONG   | NO       |
| endTimestamp   | LONG   | NO       |
| page           | INT    | NO       | default 1            |
| rows           | INT    | NO       | default 100, max 100 |
| recvWindow     | LONG   | NO       |
| timestamp      | LONG   | YES      |

* 若startTimestamp和endTimestamp均未发送,只返回最近30天数据。
* startTimestamp和endTimestamp的最大时间间隔为30天。
* 只能查询最近 6 个月的数据。如果需要产看全部C2C订单，你可以前往 https://c2c.pexpay.com/zh-CN/fiatOrder


# 错误代码

> 错误JSON格式:

```javascript
{
  "code": ${error code},
  "msg": "error message."
}
```

错误由两部分组成：错误代码和消息。 代码是通用的，但是消息可能会有所不同。 




## 10xx -常规服务器或网络问题
### -1000 UNKNOWN
 * 处理请求时发生未知错误。
 * 处理请求时发生未知错误。[%s]

### -1001 DISCONNECTED
 * 内部错误; 无法处理您的请求。 请再试一次.

### -1002 UNAUTHORIZED
 * 您无权执行此请求。

### -1003 TOO_MANY_REQUESTS
 * 排队的请求过多。
 * 请求权重过多； 请使用websocket获取实时更新。
 * 请求权重过多； 当前限制为每分钟％s请求权重。 请使用websocket进行实时更新，以避免轮询API。
 * 请求权重过多； IP被禁止，直到％s。 请使用websocket进行实时更新，以免被禁。

### -1004 SERVER_BUSY
 * 服务器正忙，请稍候再试。

### -1006 UNEXPECTED_RESP
 * 从消息总线收到意外的响应。 执行状态未知。

### -1007 TIMEOUT
 * 等待后端服务器响应超时。 发送状态未知； 执行状态未知。

### -1008 SERVER_BUSY
  * 现货交易服务器当前因其他请求而过载。 请在几分钟后重试。

### -1014 UNKNOWN_ORDER_COMPOSITION
 * 不支持的订单组合。

### -1015 TOO_MANY_ORDERS
 * 新订单太多。
 * 新订单太多； 当前限制为每％s ％s个订单。

### -1016 SERVICE_SHUTTING_DOWN
 * 该服务不可用。

### -1020 UNSUPPORTED_OPERATION
 * 不支持此操作。

### -1021 INVALID_TIMESTAMP
 * 此请求的时间戳在recvWindow之外。
 * 此请求的时间戳比服务器时间提前1000毫秒。

### -1022 INVALID_SIGNATURE
 * 此请求的签名无效。

### -1099 Not found, authenticated, or authorized
 * 替换错误代码-1999
