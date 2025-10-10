# Spin API 接口文档

## 目录
- [简介](#简介)
- [接口规范](#接口规范)
  - [GET /api/events - 查询事件数据](#get-apievents---查询事件数据)
  - [GET /api/mints - 查询所有代币](#get-apimints---查询所有代币)
  - [GET /api/mint_orders - 查询代币订单信息](#get-apimint_orders---查询代币订单信息)
  - [GET /api/user_orders - 查询用户订单信息](#get-apiuser_orders---查询用户订单信息)
  - [GET /api/user_event - 查询用户交易事件](#get-apiuser_event---查询用户交易事件)
  - [POST /api/details - 查询代币详情](#post-apidetails---查询代币详情)

## 简介

本文档详细描述了Spin API服务提供的主要接口，这些接口用于查询链上事件数据、代币信息、订单信息以及用户交易记录。所有接口均采用标准HTTP RESTful规范，返回JSON格式的数据。

## 接口规范

所有API响应均使用统一的响应格式：

```json
{
  "success": true,  // 请求是否成功
  "data": {},       // 成功时包含实际数据
  "message": ""     // 错误时包含错误信息
}
```

### GET /api/events - 查询事件数据

#### 接口描述
查询特定代币的所有相关事件，包括代币创建、买卖、做多做空、强制平仓、全部平仓和部分平仓事件。

#### 请求参数

| 参数名 | 类型 | 必填 | 说明 |
|-------|------|------|------|
| mint | string | 是 | 代币地址 |
| page | number | 否 | 页码，从1开始，默认为1 |
| limit | number | 否 | 每页记录数，最大1000，默认50 |
| order_by | string | 否 | 排序方式，可选值："slot_asc"(按区块高度升序)或"slot_desc"(按区块高度降序) |

#### 示例请求
```
GET /api/events?mint=2M5dgwGNYHAC3CQVYiriY1DYC4GETDDb3ABWv3qsx3Jr&page=1&limit=10&order_by=slot_desc
```

#### 响应参数

| 参数名 | 类型 | 说明 |
|-------|------|------|
| events | array | 事件列表 |
| total | number | 总记录数 |
| page | number | 当前页码 |
| limit | number | 每页记录数 |
| has_next | boolean | 是否有下一页 |
| has_prev | boolean | 是否有上一页 |

#### 事件类型说明

事件数据中的`event_type`字段表示事件类型，可能的值有：
- `TokenCreated`: 代币创建事件
- `BuySell`: 代币买卖事件
- `LongShort`: 做多/做空事件
- `ForceLiquidate`: 强制平仓事件
- `FullClose`: 全部平仓事件
- `PartialClose`: 部分平仓事件

不同类型的事件包含不同的数据字段，但都包含基本信息如签名、时间戳和区块高度。

##### TokenCreated事件字段说明

| 字段名 | 类型 | 说明 |
|-------|------|------|
| payer | string | 支付者地址 |
| mint_account | string | 代币地址 |
| curve_account | string | 曲线账户地址 |
| pool_token_account | string | 池代币账户地址 |
| pool_sol_account | string | 池SOL账户地址 |
| fee_recipient | string | 手续费接收者地址 |
| name | string | 代币名称 |
| symbol | string | 代币符号 |
| uri | string | 代币元数据URI |
| timestamp | string | 创建时间 |
| signature | string | 交易签名 |
| slot | number | 区块高度 |

#### 示例响应
```json
{
  "success": true,
  "data": {
    "events": [
      {
        "event_type": "TokenCreated",
        "payer": "8sYgCzLGNXmNrEr7xMpG8r6RXN7QvzJkAYNNYXxTrKHK",
        "mint_account": "2M5dgwGNYHAC3CQVYiriY1DYC4GETDDb3ABWv3qsx3Jr",
        "curve_account": "7Q3sNmgvgPTsLQyxYsK8SLNLyPyJU7dvWkQsrJJ5oGZk",
        "pool_token_account": "9J4yDqU6wBkdhP5bmJhukhsEzBkaAXiBmii52kTNL3zB",
        "pool_sol_account": "FbaCgpZpKaXJVBaGzvE2WnMYYYqQr7jtBpTerj8YTsGf",
        "fee_recipient": "4M1AQ3StRt5QSXy7ERAQ84BQyJHpNjPV5kQiHWUYAYMy",
        "name": "SpinPet Token",
        "symbol": "SPT",
        "uri": "https://example.com/token-metadata.json",
        "timestamp": "2023-10-15T08:30:45Z",
        "signature": "2Tyh7MqtiEaESWwtDjkzGZZAhqEYChxDkYJ4KK4YzDiHzR6kFRUrS2rsGuuRYhoTa46jmR1GjgCR14WXDQQMhGXr",
        "slot": 230400555
      },
      {
        "event_type": "BuySell",
        "payer": "HvwC9QSAzvGXhhVrgPmauVwFWcYZhne3hVot9EbHuFTm",
        "mint_account": "2M5dgwGNYHAC3CQVYiriY1DYC4GETDDb3ABWv3qsx3Jr",
        "is_buy": true,
        "token_amount": 5000000000,
        "sol_amount": 1000000000,
        "latest_price": "200000000",
        "timestamp": "2023-10-15T09:15:32Z",
        "signature": "5KtQ23JGJAXzYCNKNTYDLKUFVyNxuUxTYc6qrfJLWVr14DdMDTZdwGVLFaJ1sQvK9nFD7LgRnvqsmnECvb9iDPYb",
        "slot": 230405678
      }
    ],
    "total": 156,
    "page": 1,
    "limit": 10,
    "has_next": true,
    "has_prev": false
  },
  "message": ""
}
```

### GET /api/mints - 查询所有代币

#### 接口描述
查询系统中所有注册的代币列表。

#### 请求参数

| 参数名 | 类型 | 必填 | 说明 |
|-------|------|------|------|
| page | number | 否 | 页码，从1开始，默认为1 |
| limit | number | 否 | 每页记录数，最大1000，默认50 |

#### 示例请求
```
GET /api/mints?page=1&limit=20
```

#### 响应参数

| 参数名 | 类型 | 说明 |
|-------|------|------|
| mints | array | 代币地址列表 |
| total | number | 总记录数 |
| page | number | 当前页码 |
| limit | number | 每页记录数 |
| has_next | boolean | 是否有下一页 |
| has_prev | boolean | 是否有上一页 |

#### 示例响应
```json
{
  "success": true,
  "data": {
    "mints": [
      "2M5dgwGNYHAC3CQVYiriY1DYC4GETDDb3ABWv3qsx3Jr",
      "3TcTZaiCMhCDF2PM7QBzX2aHFeJqLKJrd9LFGLugkr5x",
      "8JU5yhMqvKJBYDrKZdW3TJUrTJVsxGP9Y7XJfWaRoYHF",
      "A6QoNTvZTMvg8N8NuL5VHx9www4musE3y2J9rQU7pEJ8"
    ],
    "total": 42,
    "page": 1,
    "limit": 20,
    "has_next": true,
    "has_prev": false
  },
  "message": ""
}
```

### GET /api/mint_orders - 查询代币订单信息

#### 接口描述
查询特定代币的多空订单信息。

#### 请求参数

| 参数名 | 类型 | 必填 | 说明 |
|-------|------|------|------|
| mint | string | 是 | 代币地址 |
| type | string | 是 | 订单类型，可选值："up_orders"(做空订单)或"down_orders"(做多订单) |
| page | number | 否 | 页码，从1开始，默认为1 |
| limit | number | 否 | 每页记录数，最大1000，默认50 |

#### 示例请求
```
GET /api/mint_orders?mint=2M5dgwGNYHAC3CQVYiriY1DYC4GETDDb3ABWv3qsx3Jr&type=up_orders&page=1&limit=10
```

#### 响应参数

| 参数名 | 类型 | 说明 |
|-------|------|------|
| orders | array | 订单列表 |
| total | number | 总记录数 |
| order_type | string | 查询的订单类型 |
| mint_account | string | 代币地址 |
| page | number | 当前页码 |
| limit | number | 每页记录数 |
| has_next | boolean | 是否有下一页 |
| has_prev | boolean | 是否有上一页 |

#### 订单数据字段说明

| 字段名 | 类型 | 说明 |
|-------|------|------|
| order_type | number | 订单类型，0表示做多，1表示做空 |
| mint | string | 代币地址 |
| user | string | 用户地址 |
| lock_lp_start_price | string | 锁定LP起始价格 |
| lock_lp_end_price | string | 锁定LP结束价格 |
| lock_lp_sol_amount | number | 锁定LP的SOL数量 |
| lock_lp_token_amount | number | 锁定LP的代币数量 |
| start_time | number | 订单开始时间戳 |
| end_time | number | 订单结束时间戳 |
| margin_sol_amount | number | 保证金SOL数量 |
| borrow_amount | number | 借款数量 |
| position_asset_amount | number | 仓位资产数量 |
| borrow_fee | number | 借款费率 |
| order_pda | string | 订单PDA地址 |

#### 示例响应
```json
{
  "success": true,
  "data": {
    "orders": [
      {
        "order_type": 1,
        "mint": "2M5dgwGNYHAC3CQVYiriY1DYC4GETDDb3ABWv3qsx3Jr",
        "user": "HvwC9QSAzvGXhhVrgPmauVwFWcYZhne3hVot9EbHuFTm",
        "lock_lp_start_price": "200000000",
        "lock_lp_end_price": "210000000",
        "lock_lp_sol_amount": 500000000,
        "lock_lp_token_amount": 2500000000,
        "start_time": 1697361600,
        "end_time": 1697448000,
        "margin_sol_amount": 200000000,
        "borrow_amount": 1000000000,
        "position_asset_amount": 5000000000,
        "borrow_fee": 250,
        "order_pda": "5V7NCZ7s9XRVKgYMcP4ZTQu18Y1YiyS9Vje7rDzP5X5q"
      }
    ],
    "total": 24,
    "order_type": "up_orders",
    "mint_account": "2M5dgwGNYHAC3CQVYiriY1DYC4GETDDb3ABWv3qsx3Jr",
    "page": 1,
    "limit": 10,
    "has_next": true,
    "has_prev": false
  },
  "message": ""
}
```

### GET /api/user_orders - 查询用户订单信息

#### 接口描述
查询特定用户的有效订单信息，支持按代币地址进行精确查询。该接口提供按用户维度查询订单数据的能力，支持分页和排序。

#### 请求参数

| 参数名 | 类型 | 必填 | 说明 |
|-------|------|------|------|
| user | string | 是 | 用户地址 |
| mint | string | 否 | 代币地址，如果提供则只返回该代币相关的订单 |
| page | number | 否 | 页码，从1开始，默认为1 |
| limit | number | 否 | 每页记录数，最大1000，默认50 |
| order_by | string | 否 | 排序方式，可选值："start_time_asc"(按开始时间升序)或"start_time_desc"(按开始时间降序，默认) |

#### 示例请求

**查询用户所有订单**:
```
GET /api/user_orders?user=3wsivQKRHHNX6pry2QPujQg9361ofghPeFsyCU7hZHb6&page=1&limit=50
```

**查询用户在特定代币上的订单**:
```
GET /api/user_orders?user=3wsivQKRHHNX6pry2QPujQg9361ofghPeFsyCU7hZHb6&mint=Ak5yF7oM1hE2eyXBHvypCzZhyPtgcohxXzY39s5XuyQy&order_by=start_time_desc
```

**按时间升序查询**:
```
GET /api/user_orders?user=3wsivQKRHHNX6pry2QPujQg9361ofghPeFsyCU7hZHb6&order_by=start_time_asc&page=1&limit=20
```

#### 响应参数

| 参数名 | 类型 | 说明 |
|-------|------|------|
| orders | array | 订单列表 |
| total | number | 总记录数 |
| user | string | 查询的用户地址 |
| mint_account | string | 查询的代币地址(如果提供) |
| page | number | 当前页码 |
| limit | number | 每页记录数 |
| has_next | boolean | 是否有下一页 |
| has_prev | boolean | 是否有上一页 |

#### 订单数据字段说明

| 字段名 | 类型 | 说明 |
|-------|------|------|
| order_type | number | 订单类型，1表示做多，2表示做空 |
| mint | string | 代币地址 |
| user | string | 用户地址 |
| lock_lp_start_price | string | 锁定LP起始价格 |
| lock_lp_end_price | string | 锁定LP结束价格 |
| lock_lp_sol_amount | number | 锁定LP的SOL数量 |
| lock_lp_token_amount | number | 锁定LP的代币数量 |
| start_time | number | 订单开始时间戳 |
| end_time | number | 订单结束时间戳 |
| margin_sol_amount | number | 保证金SOL数量 |
| borrow_amount | number | 借款数量 |
| position_asset_amount | number | 仓位资产数量 |
| borrow_fee | number | 借款费率 |
| order_pda | string | 订单PDA地址 |

#### 示例响应

**查询用户在特定代币上的订单**:
```json
{
  "success": true,
  "data": {
    "orders": [
      {
        "order_type": 1,
        "mint": "Ak5yF7oM1hE2eyXBHvypCzZhyPtgcohxXzY39s5XuyQy",
        "user": "3wsivQKRHHNX6pry2QPujQg9361ofghPeFsyCU7hZHb6",
        "lock_lp_start_price": "9695171046908908294242",
        "lock_lp_end_price": "9688153347722366739666",
        "lock_lp_sol_amount": 63947874,
        "lock_lp_token_amount": 65982364399,
        "start_time": 1755749556,
        "end_time": 1755922356,
        "margin_sol_amount": 36435813,
        "borrow_amount": 100000000,
        "position_asset_amount": 65982364399,
        "borrow_fee": 600,
        "order_pda": "65njgjT7J9uWVHmUgXbEr4i2g7wZpeovugyjJ2bC4tny"
      },
      {
        "order_type": 1,
        "mint": "Ak5yF7oM1hE2eyXBHvypCzZhyPtgcohxXzY39s5XuyQy",
        "user": "3wsivQKRHHNX6pry2QPujQg9361ofghPeFsyCU7hZHb6",
        "lock_lp_start_price": "9660269231515948850487",
        "lock_lp_end_price": "9513824232064239768134",
        "lock_lp_sol_amount": 1341732020,
        "lock_lp_token_amount": 1399566720549,
        "start_time": 1755749555,
        "end_time": 1755922355,
        "margin_sol_amount": 766318372,
        "borrow_amount": 2100000000,
        "position_asset_amount": 1399566720549,
        "borrow_fee": 600,
        "order_pda": "3FUP3adtAqY1cjmhrnuaehTF6unPmuTk4HZaHzoWjwJt"
      }
    ],
    "total": 2,
    "user": "3wsivQKRHHNX6pry2QPujQg9361ofghPeFsyCU7hZHb6",
    "mint_account": "Ak5yF7oM1hE2eyXBHvypCzZhyPtgcohxXzY39s5XuyQy",
    "page": 1,
    "limit": 50,
    "has_next": false,
    "has_prev": false
  },
  "message": "Operation successful"
}
```

**查询用户所有订单（不指定代币）**:
```json
{
  "success": true,
  "data": {
    "orders": [
      {
        "order_type": 1,
        "mint": "Ak5yF7oM1hE2eyXBHvypCzZhyPtgcohxXzY39s5XuyQy",
        "user": "3wsivQKRHHNX6pry2QPujQg9361ofghPeFsyCU7hZHb6",
        "lock_lp_start_price": "9695171046908908294242",
        "lock_lp_end_price": "9688153347722366739666",
        "lock_lp_sol_amount": 63947874,
        "lock_lp_token_amount": 65982364399,
        "start_time": 1755749556,
        "end_time": 1755922356,
        "margin_sol_amount": 36435813,
        "borrow_amount": 100000000,
        "position_asset_amount": 65982364399,
        "borrow_fee": 600,
        "order_pda": "65njgjT7J9uWVHmUgXbEr4i2g7wZpeovugyjJ2bC4tny"
      },
      {
        "order_type": 2,
        "mint": "Bk6yG8pN2iF3fzYCIwzqD0aA1bJgdpjixYzY40t6vzRz",
        "user": "3wsivQKRHHNX6pry2QPujQg9361ofghPeFsyCU7hZHb6",
        "lock_lp_start_price": "9550000000000000000000",
        "lock_lp_end_price": "9450000000000000000000",
        "lock_lp_sol_amount": 1000000000,
        "lock_lp_token_amount": 1000000000000,
        "start_time": 1755749000,
        "end_time": 1755921800,
        "margin_sol_amount": 500000000,
        "borrow_amount": 1500000000,
        "position_asset_amount": 1000000000000,
        "borrow_fee": 500,
        "order_pda": "4GVP4beuBrZ2dkmisobfUG7voQnvQnvTk5IabIapXkK"
      }
    ],
    "total": 2,
    "user": "3wsivQKRHHNX6pry2QPujQg9361ofghPeFsyCU7hZHb6",
    "mint_account": null,
    "page": 1,
    "limit": 50,
    "has_next": false,
    "has_prev": false
  },
  "message": "Operation successful"
}
```

#### 使用场景

1. **用户订单管理**: 查询用户的所有有效订单，用于订单管理界面
2. **代币特定查询**: 查询用户在特定代币上的订单，用于代币详情页面
3. **订单分析**: 按时间排序查询订单，用于分析用户的交易行为
4. **分页展示**: 支持大量订单数据的分页展示

#### 注意事项

- 该接口只返回用户当前有效的订单数据
- 订单数据按 `start_time` 字段排序，默认降序（最新的在前）
- 当指定 `mint` 参数时，`mint_account` 字段会返回该代币地址
- 当不指定 `mint` 参数时，`mint_account` 字段为 `null`

### GET /api/user_event - 查询用户交易事件

#### 接口描述
查询特定用户的交易事件记录，可选择性过滤特定代币的交易。

#### 请求参数

| 参数名 | 类型 | 必填 | 说明 |
|-------|------|------|------|
| user | string | 是 | 用户地址 |
| mint | string | 否 | 代币地址，如果提供则只返回该代币相关的交易 |
| page | number | 否 | 页码，从1开始，默认为1 |
| limit | number | 否 | 每页记录数，最大1000，默认50 |
| order_by | string | 否 | 排序方式，可选值："slot_asc"(按区块高度升序)或"slot_desc"(按区块高度降序) |

#### 示例请求
```
GET /api/user_event?user=HvwC9QSAzvGXhhVrgPmauVwFWcYZhne3hVot9EbHuFTm&page=1&limit=10&order_by=slot_desc
```

#### 响应参数

| 参数名 | 类型 | 说明 |
|-------|------|------|
| transactions | array | 交易事件列表 |
| total | number | 总记录数 |
| page | number | 当前页码 |
| limit | number | 每页记录数 |
| has_next | boolean | 是否有下一页 |
| has_prev | boolean | 是否有上一页 |
| user | string | 查询的用户地址 |
| mint_account | string | 查询的代币地址(如果提供) |

#### 用户交易数据字段说明

| 字段名 | 类型 | 说明 |
|-------|------|------|
| event_type | string | 事件类型，可能的值有："long_short", "force_liquidate", "full_close", "partial_close" |
| user | string | 用户地址 |
| mint_account | string | 代币地址 |
| slot | number | 区块高度 |
| timestamp | number | 时间戳 |
| signature | string | 交易签名 |
| event_data | object | 完整的事件数据，根据事件类型有不同的结构 |

#### 示例响应
```json
{
  "success": true,
  "data": {
    "transactions": [
      {
        "event_type": "long_short",
        "user": "HvwC9QSAzvGXhhVrgPmauVwFWcYZhne3hVot9EbHuFTm",
        "mint_account": "2M5dgwGNYHAC3CQVYiriY1DYC4GETDDb3ABWv3qsx3Jr",
        "slot": 230405900,
        "timestamp": 1697369732,
        "signature": "5V7NCZ7s9XRVKgYMcP4ZTQu18Y1YiyS9Vje7rDzP5X5q",
        "event_data": {
          "payer": "HvwC9QSAzvGXhhVrgPmauVwFWcYZhne3hVot9EbHuFTm",
          "mint_account": "2M5dgwGNYHAC3CQVYiriY1DYC4GETDDb3ABWv3qsx3Jr",
          "order_pda": "5V7NCZ7s9XRVKgYMcP4ZTQu18Y1YiyS9Vje7rDzP5X5q",
          "latest_price": "200000000",
          "order_type": 1,
          "mint": "2M5dgwGNYHAC3CQVYiriY1DYC4GETDDb3ABWv3qsx3Jr",
          "user": "HvwC9QSAzvGXhhVrgPmauVwFWcYZhne3hVot9EbHuFTm",
          "lock_lp_start_price": "200000000",
          "lock_lp_end_price": "210000000",
          "lock_lp_sol_amount": 500000000,
          "lock_lp_token_amount": 2500000000,
          "start_time": 1697361600,
          "end_time": 1697448000,
          "margin_sol_amount": 200000000,
          "borrow_amount": 1000000000,
          "position_asset_amount": 5000000000,
          "borrow_fee": 250,
          "timestamp": "2023-10-15T11:28:52Z",
          "signature": "5V7NCZ7s9XRVKgYMcP4ZTQu18Y1YiyS9Vje7rDzP5X5q",
          "slot": 230405900
        }
      },
      {
        "event_type": "full_close",
        "user": "HvwC9QSAzvGXhhVrgPmauVwFWcYZhne3hVot9EbHuFTm",
        "mint_account": "2M5dgwGNYHAC3CQVYiriY1DYC4GETDDb3ABWv3qsx3Jr",
        "slot": 230415800,
        "timestamp": 1697376332,
        "signature": "3xqT5MeAU6Jd7VRVpvjxbYQQRmcYW9bfVsyxXLqvGBnMSK5QSw72nCWNJGaQTC2WecD6vHvPTD5xXmsyHmwt9Nu6",
        "event_data": {
          "payer": "HvwC9QSAzvGXhhVrgPmauVwFWcYZhne3hVot9EbHuFTm",
          "user_sol_account": "Eh5jw9rSpxkbwHEJ6Sw3EmiRm4qKXHxVnKShTGPTbLJi",
          "mint_account": "2M5dgwGNYHAC3CQVYiriY1DYC4GETDDb3ABWv3qsx3Jr",
          "is_close_long": false,
          "final_token_amount": 5200000000,
          "final_sol_amount": 220000000,
          "user_close_profit": 20000000,
          "latest_price": "230000000",
          "order_pda": "5V7NCZ7s9XRVKgYMcP4ZTQu18Y1YiyS9Vje7rDzP5X5q",
          "timestamp": "2023-10-15T13:18:52Z",
          "signature": "3xqT5MeAU6Jd7VRVpvjxbYQQRmcYW9bfVsyxXLqvGBnMSK5QSw72nCWNJGaQTC2WecD6vHvPTD5xXmsyHmwt9Nu6",
          "slot": 230415800
        }
      }
    ],
    "total": 8,
    "page": 1,
    "limit": 10,
    "has_next": false,
    "has_prev": false,
    "user": "HvwC9QSAzvGXhhVrgPmauVwFWcYZhne3hVot9EbHuFTm",
    "mint_account": null
  },
  "message": ""
}
```

### POST /api/details - 查询代币详情

#### 接口描述
查询多个代币的详细信息，包括创建信息、价格信息和统计数据。

#### 请求参数

| 参数名 | 类型 | 必填 | 说明 |
|-------|------|------|------|
| mints | array | 是 | 要查询的代币地址列表 |

#### 示例请求
```json
{
  "mints": [
    "2M5dgwGNYHAC3CQVYiriY1DYC4GETDDb3ABWv3qsx3Jr",
    "3TcTZaiCMhCDF2PM7QBzX2aHFeJqLKJrd9LFGLugkr5x"
  ]
}
```

#### 响应参数

| 参数名 | 类型 | 说明 |
|-------|------|------|
| details | array | 代币详情列表 |
| total | number | 总记录数 |

#### 代币详情数据字段说明

| 字段名 | 类型 | 说明 |
|-------|------|------|
| mint_account | string | 代币地址 |
| payer | string | 创建者地址 |
| curve_account | string | 曲线账户地址 |
| pool_token_account | string | 池代币账户地址 |
| pool_sol_account | string | 池SOL账户地址 |
| fee_recipient | string | 手续费接收者地址 |
| name | string | 代币名称 |
| symbol | string | 代币符号 |
| uri | string | 代币元数据URI |
| swap_fee | number | 现货交易手续费 |
| borrow_fee | number | 保证金交易手续费 |
| fee_discount_flag | number | 手续费折扣标志（0:原价 1:5折 2:2.5折 3:1.25折）|
| create_timestamp | number | 创建时间戳 |
| latest_price | string | 最新价格 |
| latest_trade_time | number | 最近交易时间戳 |
| total_sol_amount | number | 总SOL数量 |
| total_margin_sol_amount | number | 总保证金SOL数量 |
| total_force_liquidations | number | 总强制平仓次数 |
| total_close_profit | number | 总平仓收益 |

#### 示例响应
```json
{
  "success": true,
  "data": {
    "details": [
      {
        "mint_account": "2M5dgwGNYHAC3CQVYiriY1DYC4GETDDb3ABWv3qsx3Jr",
        "payer": "8sYgCzLGNXmNrEr7xMpG8r6RXN7QvzJkAYNNYXxTrKHK",
        "curve_account": "7Q3sNmgvgPTsLQyxYsK8SLNLyPyJU7dvWkQsrJJ5oGZk",
        "pool_token_account": "9J4yDqU6wBkdhP5bmJhukhsEzBkaAXiBmii52kTNL3zB",
        "pool_sol_account": "FbaCgpZpKaXJVBaGzvE2WnMYYYqQr7jtBpTerj8YTsGf",
        "fee_recipient": "4M1AQ3StRt5QSXy7ERAQ84BQyJHpNjPV5kQiHWUYAYMy",
        "name": "SpinPet Token",
        "symbol": "SPT",
        "uri": "https://example.com/token-metadata.json",
        "swap_fee": 100,
        "borrow_fee": 200,
        "fee_discount_flag": 0,
        "create_timestamp": 1697356245,
        "latest_price": "220000000",
        "latest_trade_time": 1697376332,
        "total_sol_amount": 5200000000,
        "total_margin_sol_amount": 200000000,
        "total_force_liquidations": 0,
        "total_close_profit": 20000000
      }
    ],
    "total": 1
  },
  "message": ""
}
```
