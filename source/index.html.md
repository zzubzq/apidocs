---
title: Pexpay API Documentation
language_tabs: # must be one of https://git.io/vQNgJ
  #- shell
  #- javascript
  #- json

toc_footers:
  - <a href='https://www.pexpay.com/'>Pexpay Exchange</a>

includes:

search: true

---


# Change Log
<font size=4>**2022-11-07**</font>

* GET /api/v1/aggTrades 
* GET /api/v1/klines 
* GET /api/v3/order 
* GET /api/v3/allOrders 
* GET /api/v3/account 
* GET /api/v3/myTrades 
* GET /api/v3/historicalTrades 



# Introduction

## API Key Setup

* Some endpoints will require an API Key. Please refer to [this page](https://www.pexpayzh.top/en/support/faq/360002502072) regarding API key creation.
* Once API key is created, it is recommended to set IP restrictions on the key for security reasons.
* **Never share your API key/secret key to ANYONE.**

<aside class="warning">
If the API keys were accidentally shared, please delete them immediately and create a new key.
</aside>



## Contact Us

* [submit your requests or issues in here](https://support.pexpay.com/hc/en-us/requests/new?ticket_form_id=11868568981785)

---

# General Info
## General API Information

* The base endpoint is: **https://api.pexpay.com**
* All endpoints return either a JSON object or array.
* All time and timestamp related fields are in **milliseconds**.

### HTTP Return Codes
* HTTP `4XX` return codes are used for malformed requests;
  the issue is on the sender's side.
* HTTP `403` return code is used when the WAF Limit (Web Application Firewall) has been violated.
* HTTP `429` return code is used when breaking a request rate limit.
* HTTP `418` return code is used when an IP has been auto-banned for continuing to send requests after receiving `429` codes.
* HTTP `5XX` return codes are used for internal errors; the issue is on
  Pexpay's side.
  It is important to **NOT** treat this as a failure operation; the execution status is
  **UNKNOWN** and could have been a success.

### Error Codes and Messages

* If there is an error, the API will return an error with a message of the reason.

> The error payload is as follows:
 
```javascript
{
  "code": ${error code},
  "msg": "error message."
}
```

* Specific error codes and messages defined in [Error Codes](#error-codes).

### General Information on Endpoints

* For `GET` endpoints, parameters must be sent as a `query string`.
* For `POST`, `PUT`, and `DELETE` endpoints, the parameters may be sent as a
  `query string` or in the `request body` with content type
  `application/x-www-form-urlencoded`. You may mix parameters between both the
  `query string` and `request body` if you wish to do so.
* Parameters may be sent in any order.
* If a parameter sent in both the `query string` and `request body`, the
  `query string` parameter will be used.

---
## LIMITS

### General Info on Limits
* The following `intervalLetter` values for headers:
    * SECOND => S
    * MINUTE => M
    * HOUR => H
    * DAY => D
* `intervalNum` describes the amount of the interval. For example, `intervalNum` 5 with `intervalLetter` M means "Every 5 minutes".
* A 429 will be returned when either rate limit is violated.

### IP Limits
* Every request will contain `X-MBX-USED-WEIGHT-(intervalNum)(intervalLetter)` in the response headers which has the current used weight for the IP for all request rate limiters defined.
* Each route has a `weight` which determines for the number of requests each endpoint counts for. Heavier endpoints and endpoints that do operations on multiple symbols will have a heavier `weight`.
* When a 429 is received, it's your obligation as an API to back off and not spam the API.
* **Repeatedly violating rate limits and/or failing to back off after receiving 429s will result in an automated IP ban (HTTP status 418).**
* IP bans are tracked and **scale in duration** for repeat offenders, **from 2 minutes to 3 days**.
* A `Retry-After` header is sent with a 418 or 429 responses and will give the **number of seconds** required to wait, in the case of a 429, to prevent a ban, or, in the case of a 418, until the ban is over.
* **The limits on the API are based on the IPs, not the API keys.**


### /sapi/ Limit Introduction

The `/sapi/*` endpoints adopt either of two access limiting rules, IP limits or UID (account) limits.

  * Endpoints are marked according to IP or UID limit and their corresponding weight value.
  * Each endpoint with IP limits has an independent 12000 per minute limit.
  * Each endpoint with UID limits has an independent 180000 per minute limit.
  * Responses from endpoints with IP limits contain the header `X-SAPI-USED-IP-WEIGHT-1M`, defining the weight used by the current IP.
  * Responses from endpoints with UID limits contain the header `X-SAPI-USED-UID-WEIGHT-1M`, defining the weight used by the current UID.

---
## Endpoint security type
* Each endpoint has a security type that determines how you will
  interact with it. This is stated next to the NAME of the endpoint.
    * If no security type is stated, assume the security type is NONE.
* API-keys are passed into the Rest API via the `X-MBX-APIKEY`
  header.
* API-keys and secret-keys **are case sensitive**.
* API-keys can be configured to only access certain types of secure endpoints.
 For example, one API-key could be used for TRADE only, while another API-key
 can access everything except for TRADE routes.
* By default, API-keys can access all secure routes.

| Security Type | Description                                              |
| ------------- | -------------------------------------------------------- |
| NONE          | Endpoint can be accessed freely.                         |
| TRADE         | Endpoint requires sending a valid API-Key and signature. |
| USER_DATA     | Endpoint requires sending a valid API-Key and signature. |



* `TRADE` and `USER_DATA` endpoints are `SIGNED` endpoints.

---
## SIGNED (TRADE, USER_DATA, AND MARGIN) Endpoint security
* `SIGNED` endpoints require an additional parameter, `signature`, to be
  sent in the  `query string` or `request body`.
* Endpoints use `HMAC SHA256` signatures. The `HMAC SHA256 signature` is a keyed `HMAC SHA256` operation.
  Use your `secretKey` as the key and `totalParams` as the value for the HMAC operation.
* The `signature` is **not case sensitive**.
* `totalParams` is defined as the `query string` concatenated with the
  `request body`.

### Timing security
* A `SIGNED` endpoint also requires a parameter, `timestamp`, to be sent which
  should be the millisecond timestamp of when the request was created and sent.
* An additional parameter, `recvWindow`, may be sent to specify the number of
  milliseconds after `timestamp` the request is valid for. If `recvWindow`
  is not sent, **it defaults to 5000**.

> The logic is as follows:

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

**Serious trading is about timing.** Networks can be unstable and unreliable,
which can lead to requests taking varying amounts of time to reach the
servers. With `recvWindow`, you can specify that the request must be
processed within a certain number of milliseconds or be rejected by the
server.

<aside class="notice">
It is recommended to use a small recvWindow of 5000 or less! The max cannot go beyond 60,000!
</aside>

---
### SIGNED Endpoint Examples for POST /sapi/v1/c2c/ads/getDetailByNo 
Here is a step-by-step example of how to send a vaild signed payload from the
Linux command line using `echo`, `openssl`, and `curl`.

| Key       | Value                                                            |
| --------- | ---------------------------------------------------------------- |
| apiKey    | dbefbc809e3e83c283a984c3a1459732ea7db1360ca80c5c2c8867408d28cc83 |
| secretKey | 2b5eb11e18796d12d88f13dc27dbbd02c2cc51ff7059765ed9821957d82bb4d9 |


| Parameter        | Value          |
| ----------- | ------------- |
| adsNo       | 10191633467710386176        |
| timestamp        | 1591702613943           |



#### Example 1: As a request body

> **Example 1**

> **HMAC SHA256 signature:**

```shell
    $ echo -n "adsNo=10191633467710386176&timestamp=1591702613943" | openssl dgst -sha256 -hmac "2b5eb11e18796d12d88f13dc27dbbd02c2cc51ff7059765ed9821957d82bb4d9"
    (stdin)=  3c661234138461fcc7a7d8746c6558c9842d4e10870d2ecbedf7777cad694af9
```


> **curl command:**

```shell
    (HMAC SHA256)
    $ curl -H "X-MBX-APIKEY: dbefbc809e3e83c283a984c3a1459732ea7db1360ca80c5c2c8867408d28cc83" -X POST 'https://api.pexpay.com/sapi/v1/c2c/ads/getDetailByNo' -d 'sadsNo=10191633467710386176&timestamp=1591702613943&signature=3c661234138461fcc7a7d8746c6558c9842d4e10870d2ecbedf7777cad694af9'
    
```


#### 示例 2: Example 2: As a query string

> **Example 2**

> **HMAC SHA256 signature:**

```shell
    $ echo -n "adsNo=10191633467710386176&timestamp=1591702613943" | openssl dgst -sha256 -hmac "2b5eb11e18796d12d88f13dc27dbbd02c2cc51ff7059765ed9821957d82bb4d9"
    (stdin)= 3c661234138461fcc7a7d8746c6558c9842d4e10870d2ecbedf7777cad694af9
    
```
> **curl command:**

```shell
    (HMAC SHA256)
   $ curl -H "X-MBX-APIKEY: dbefbc809e3e83c283a984c3a1459732ea7db1360ca80c5c2c8867408d28cc83" -X POST 'https://api.pexpay.com/sapi/v1/c2c/ads/getDetailByNo?adsNo=10191633467710386176&timestamp=1591702613943&signature=3c661234138461fcc7a7d8746c6558c9842d4e10870d2ecbedf7777cad694af9'
    
```

---

## Public API Definitions
### Terminology

These terms will be used throughout the documentation, so it is recommended especially for new users to read to help their understanding of the API.

* `ad` the advertisement published by maker
* `order` the order placed by taker

### ENUM definitions
**order status (c2c order related):**

| Status              | Description                                                                                                                                |
| ------------------| ----------------------------------------------------------------------------------------------------------------------------------- |
| `1`               | pending payment
| `2`               | Paid                                                                                                                          |
| `3`               | Waiting for release(system abnormal)                                                                                                                          |
| `4`               | Confirmed, Order complete                                                                                               |
| `5`               | Appealing                                                                                                    |
| `6`               | Canceled                                                                                                                           |
| `7`               | Timeout                                                                                                             |

**reason code for canceling orders (c2c order related):**


| Status              | Description                                                                                                                                |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------- |
| `1`               | I do not want to trade anymore                                                                                                                           |
| `2`               | I do not meet the requirements of the advertiser's trading terms and condition                                                                                                                      |
| `3`               | Seller is asking for extra fee                                                                                                                        |
| `4`               | Problem with seller's payment method result in unsuccessful payments                                                                                                                               | 
| `5`               | Other reasons                                                                                                                                  |
**ad status (c2c ad related):**

| Status               | Description                                                                                                                                |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------- |
| `1`               | active                                                                                                                              |
| `3`               | inactive                                                                                                                              |
| `4`               | closed                                                                                                                              |



**ad priceType (c2c ad related):**

| Status               | Description                                                                                                                                |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------- |
| `1`               | fixed price                                                                                                                              |
| `2`               | floating price                                                                                                                |




# C2C Endpoints


## Get C2C Trade History (USER_DATA)

> **Response:**

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
      "amount": "5000.00000000",  // Quantity (in Product)
      "totalPrice": "33400.00000000",
      "unitPrice": "6.68", // Unit Price (in Fiat)
      "orderStatus": "COMPLETED",  // PENDING, TRADING, BUYER_PAYED, DISTRIBUTING, COMPLETED, IN_APPEAL, CANCELLED, CANCELLED_BY_SYSTEM
      "createTime": 1619361369000,
      "commission": "0",   // Transaction Fee (in Product)
      "counterPartNickName": "ab***",
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


**Parameters:**

| Name           | Type   | Mandatory | Description          |
| -------------- | ------ | --------- | -------------------- |
| tradeType      | STRING | YES       | BUY, SELL            |
| startTimestamp | LONG   | NO        |
| endTimestamp   | LONG   | NO        |
| page           | INT    | NO        | default 1            |
| rows           | INT    | NO        | default 100, max 100 |
| recvWindow     | LONG   | NO        |
| timestamp      | LONG   | YES       |

* If startTimestamp and endTimestamp are not sent, the recent 30-day data will be returned.
* The max interval between startTimestamp and endTimestamp is 30 days.
* Only the last 6 months of data can be retrieved. To view the complete P2P order history, you can download it from https://c2c.pexpay.com/en/fiatOrder


## 获取 C2C 当前用户支持发布的广告类别   (NONE)

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
GET /sapi/v1/c2c/ads/getAvailableAdsCategory
``

**参数:**
* 不需要额外参数


## 获取 C2C 广告详情 (NONE)

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
POST /sapi/v1/c2c/ads/getDetailByNo 
``



**参数:**

| 名称           |      传参形式| 类型   | 是否必需 | 描述                 |
| --------------|-------------| ------ | -------- | -------------------- |
| adsNo         |query string | String   | YES      | 广告编号            |


> **请求示例:**

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


## 获取 C2C 广告报价 (NONE)

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
POST /sapi/v1/c2c/ads/getReferencePrice
``



**参数:**

| 名称           |      传参形式| 类型   | 是否必需 | 描述                 |
| --------------|-------------| ------ | -------- | -------------------- |
| assets         |body | List String   | YES      | 数字货币种类            |
| fiatCurrency   |body | String   | YES      | 法币            |
| payType        |body | String   | YES      | 支付方式类型，比如 BANK，WECHAT       |
| tradeType      |body | List String   | YES      | 交易类型，BUY, SELL            |



> **请求示例:**

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


## 获取 C2C 用户广告列表 (USER_DATA)

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
POST /sapi/v1/c2c/ads/listWithPagination
``



**参数:**

| 名称           |      传参形式| 类型   | 是否必需 | 描述                 |
| --------------|-------------  | ------ | -------- | -------------------- |
| advNo         |body           |  String           | YES      | 数字货币种类                   |
| advStatus     |body           | String            | YES      | 广告状态 1在线 3下线 4关闭        |
| asset         |body           | String            | YES      | 数字货币类型，如BTC  |
| startDate     |body           | String            | YES      | 广告创建开始时间            |
| endDate       |body           | String            | YES      | 广告创建结束时间            |
| fiatUnit      |body           | String            | YES      | 法币类型            |
| page          |body           | String            | YES      | 页数            |
| rows          |body           | String            | YES      | 每页条目数            |
| tradeType     |body           | List String       | YES      | 交易类型，BUY, SELL            |



> **请求示例:**

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

## 获取 C2C 发布广告 (TRADING)

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
POST /sapi/v1/c2c/ads/post
``


**参数:**

| 名称           |      传参形式| 类型   | 是否必需 | 描述                 |
| --------------|-------------  | ------ | -------- | -------------------- |
| advNo         |body           |  String           | YES      | 数字货币种类                   |
| advStatus     |body           | String            | YES      | 广告状态 1在线 3下线 4关闭        |
| asset         |body           | String            | YES      | 数字货币类型，如BTC  |
| startDate     |body           | String            | YES      | 广告创建开始时间            |
| endDate       |body           | String            | YES      | 广告创建结束时间            |
| fiatUnit      |body           | String            | YES      | 法币类型            |
| page          |body           | String            | YES      | 页数            |
| rows          |body           | String            | YES      | 每页条目数            |
| tradeType     |body           | List String       | YES      | 交易类型，BUY, SELL            |




> **请求示例:**

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


## 获取 C2C 搜索广告 (NONE)

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
POST /sapi/v1/c2c/ads/search
``


**参数:**

| 名称           |      传参形式| 类型   | 是否必需 | 描述                 |
| --------------|-------------  | ------ | -------- | -------------------- |
| advNo         |body           |  String           | YES      | 数字货币种类                   |
| advStatus     |body           | String            | YES      | 广告状态 1在线 3下线 4关闭        |
| asset         |body           | String            | YES      | 数字货币类型，如BTC  |
| startDate     |body           | String            | YES      | 广告创建开始时间            |
| endDate       |body           | String            | YES      | 广告创建结束时间            |
| fiatUnit      |body           | String            | YES      | 法币类型            |
| page          |body           | String            | YES      | 页数            |
| rows          |body           | String            | YES      | 每页条目数            |
| tradeType     |body           | List String       | YES      | 交易类型，BUY, SELL            |




> **请求示例:**

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

## 获取 C2C 更新广告状态 (TRADING)

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
POST /sapi/v1/c2c/ads/update
``


**参数:**

| 名称           |      传参形式| 类型   | 是否必需 | 描述                 |
| --------------|-------------  | ------ | -------- | -------------------- |
| advNo         |body           |  String           | YES      | 数字货币种类                   |
| advStatus     |body           | String            | YES      | 广告状态 1在线 3下线 4关闭        |
| asset         |body           | String            | YES      | 数字货币类型，如BTC  |
| startDate     |body           | String            | YES      | 广告创建开始时间            |
| endDate       |body           | String            | YES      | 广告创建结束时间            |
| fiatUnit      |body           | String            | YES      | 法币类型            |
| page          |body           | String            | YES      | 页数            |
| rows          |body           | String            | YES      | 每页条目数            |
| tradeType     |body           | List String       | YES      | 交易类型，BUY, SELL            |


 
# Error Codes

> The error JSON payload:
 
```javascript
{
  "code": ${error code},
  "msg": "error message."
}
```

Errors consist of two parts: an error code and a message. Codes are universal, but messages can vary. 

## 10xx - General Server or Network issues
### -1000 UNKNOWN
 * An unknown error occurred while processing the request.
 * An unknown error occurred while processing the request.[%s]

### -1001 DISCONNECTED
 * Internal error; unable to process your request. Please try again.

### -1002 UNAUTHORIZED
 * You are not authorized to execute this request.

### -1003 TOO_MANY_REQUESTS
 * Too many requests queued.
 * Too much request weight used; please use the websocket for live updates to avoid polling the API.
 * Too much request weight used; current limit is %s request weight per %s %s. Please use the websocket for live updates to avoid polling the API.
 * Way too much request weight used; IP banned until %s. Please use the websocket for live updates to avoid bans.

### -1004 SERVER_BUSY
 * Server is busy, please wait and try again
 
### -1006 UNEXPECTED_RESP
 * An unexpected response was received from the message bus. Execution status unknown.

### -1007 TIMEOUT
 * Timeout waiting for response from backend server. Send status unknown; execution status unknown.

### -1008 SERVER_BUSY
  * Spot server is currently overloaded with other requests. Please try again in a few minutes. 

### -1014 UNKNOWN_ORDER_COMPOSITION
 * Unsupported order combination.

### -1015 TOO_MANY_ORDERS
 * Too many new orders.
 * Too many new orders; current limit is %s orders per %s.

### -1016 SERVICE_SHUTTING_DOWN
 * This service is no longer available.

### -1020 UNSUPPORTED_OPERATION
 * This operation is not supported.

### -1021 INVALID_TIMESTAMP
 * Timestamp for this request is outside of the recvWindow.
 * Timestamp for this request was 1000ms ahead of the server's time.

### -1022 INVALID_SIGNATURE
 * Signature for this request is not valid.

### -1099 Not found, authenticated, or authorized
 * This replaces error code -1999









 







 
