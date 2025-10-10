# WebSocket 交易事件订阅 API 文档

## 概述

Spin Server 提供了通过 Socket.IO WebSocket 实时订阅交易事件的功能。当用户订阅K线数据时，除了接收K线更新外，还会同时接收该 mint 的所有交易事件数据。

## 连接信息

- **WebSocket URL**: `ws://localhost:5051/kline`
- **命名空间**: `/kline`
- **协议**: Socket.IO
- **支持的传输方式**: WebSocket, Polling

## 订阅流程

### 1. 建立连接

```javascript
const { io } = require('socket.io-client');

const socket = io('http://localhost:5051/kline', {
    transports: ['websocket', 'polling'],
    timeout: 20000,
    reconnection: true,
    reconnectionAttempts: 10,
    reconnectionDelay: 2000,
});
```

### 2. 监听连接成功

```javascript
socket.on('connect', () => {
    console.log('WebSocket 连接成功');
    console.log('Socket ID:', socket.id);
});

socket.on('connection_success', (data) => {
    console.log('连接确认:', data);
    // 响应格式:
    // {
    //   "client_id": "socket_id",
    //   "server_time": 1758805615,
    //   "supported_intervals": ["s1", "s30", "m5"],
    //   "supported_symbols": []
    // }
});
```

### 3. 订阅交易事件

要接收交易事件，需要通过订阅K线来触发：

```javascript
socket.emit('subscribe', {
    symbol: 'mint_account_address',  // mint 地址
    interval: 's30',                 // K线间隔 (s1, s30, m5)
    subscription_id: 'your_unique_id' // 可选的订阅ID
});
```

### 4. 订阅确认

```javascript
socket.on('subscription_confirmed', (data) => {
    console.log('订阅成功:', data);
    // 响应格式:
    // {
    //   "symbol": "mint_account_address",
    //   "interval": "s30",
    //   "subscription_id": "your_unique_id",
    //   "success": true,
    //   "message": "订阅成功"
    // }
});
```

## 接收事件数据

### 历史事件数据 (history_event_data)

订阅成功后，会立即收到该 mint 的历史交易事件（最多300条，按 slot 从大到小排序）：

```javascript
socket.on('history_event_data', (data) => {
    console.log('历史事件数据:', data);
});
```

**响应格式:**
```json
{
    "symbol": "mint_account_address",
    "data": [
        {
            "symbol": "mint_account_address",
            "event_type": "EventType",
            "event_data": { /* 完整事件数据 */ },
            "timestamp": 1758805615000
        }
    ],
    "has_more": false,
    "total_count": 5
}
```

### 实时事件数据 (event_data)

每当有新的交易事件发生时，会实时推送：

```javascript
socket.on('event_data', (data) => {
    console.log('实时事件:', data);
});
```

**响应格式:**
```json
{
    "symbol": "mint_account_address",
    "event_type": "EventType",
    "event_data": { /* 完整事件数据 */ },
    "timestamp": 1758805615000
}
```

## 支持的事件类型

### 1. TokenCreated - 代币创建事件

**事件类型**: `"TokenCreated"`

**数据格式:**
```json
{
    "event_type": "TokenCreated",
    "payer": "GKQXMqnr5kquB5KSngKt7gaY5yUmpp57PtDoTyqU",
    "mint_account": "4wuPEpuJXtoaJWEXmiSc4gN4nuGvcwx7dUGP8LHaMfvd",
    "curve_account": "CurveAccountPubkey",
    "pool_token_account": "PoolTokenAccountPubkey",
    "pool_sol_account": "PoolSolAccountPubkey",
    "fee_recipient": "FeeRecipientPubkey",
    "base_fee_recipient": "BaseFeeRecipientPubkey",
    "params_account": "ParamsAccountPubkey",
    "name": "My Token",
    "symbol": "MYTKN",
    "uri": "https://ipfs.io/ipfs/QmHashExample",
    "swap_fee": 100,
    "borrow_fee": 200,
    "fee_discount_flag": 0,
    "timestamp": "2025-09-25T12:41:28.507741179Z",
    "signature": "TransactionSignature",
    "slot": 1330
}
```

**字段说明:**
- `payer`: 创建者地址
- `mint_account`: 代币mint地址
- `curve_account`: 曲线账户地址
- `pool_token_account`: 池子代币账户
- `pool_sol_account`: 池子SOL账户
- `fee_recipient`: 手续费接收地址
- `base_fee_recipient`: 基础手续费接收地址
- `params_account`: 合作伙伴参数账户PDA地址
- `name`: 代币名称
- `symbol`: 代币符号
- `uri`: 代币元数据URI
- `swap_fee`: 现货交易手续费
- `borrow_fee`: 保证金交易手续费
- `fee_discount_flag`: 手续费折扣标志 (0: 原价, 1: 5折, 2: 2.5折, 3: 1.25折)

### 2. BuySell - 买卖交易事件

**事件类型**: `"BuySell"`

**数据格式:**
```json
{
    "event_type": "BuySell",
    "payer": "GKQXMqnr5kquB5KSngKt7gaY5yUmpp57PtDoTyqU",
    "mint_account": "4wuPEpuJXtoaJWEXmiSc4gN4nuGvcwx7dUGP8LHaMfvd",
    "is_buy": true,
    "token_amount": 1000000000,
    "sol_amount": 500000000,
    "latest_price": "279589934762348555452",
    "timestamp": "2025-09-25T12:41:28.507741179Z",
    "signature": "TransactionSignature",
    "slot": 1331
}
```

**字段说明:**
- `payer`: 交易者地址
- `mint_account`: 代币mint地址
- `is_buy`: 是否为买入操作 (true: 买入, false: 卖出)
- `token_amount`: 代币数量
- `sol_amount`: SOL数量 (lamports)
- `latest_price`: 最新价格 (28位精度)

### 3. LongShort - 开仓事件

**事件类型**: `"LongShort"`

**数据格式:**
```json
{
    "event_type": "LongShort",
    "payer": "GKQXMqnr5kquB5KSngKt7gaY5yUmpp57PtDoTyqU",
    "mint_account": "4wuPEpuJXtoaJWEXmiSc4gN4nuGvcwx7dUGP8LHaMfvd",
    "order_pda": "DYSd91e5qcxRuAdu6f7mPE2Sx6xuGMqQRz4FENJ494Vq",
    "latest_price": "279589934762348555452",
    "order_type": 2,
    "mint": "4wuPEpuJXtoaJWEXmiSc4gN4nuGvcwx7dUGP8LHaMfvd",
    "user": "GKQXMqnr5kquB5KSngKt7gaY5yUmpp57PtDoTyqU",
    "lock_lp_start_price": "250000000000000000000",
    "lock_lp_end_price": "300000000000000000000",
    "lock_lp_sol_amount": 1000000000,
    "lock_lp_token_amount": 500000000,
    "start_time": 1758805200,
    "end_time": 1758805800,
    "margin_sol_amount": 500000000,
    "borrow_amount": 1500000000,
    "position_asset_amount": 2000000000,
    "borrow_fee": 250,
    "timestamp": "2025-09-25T12:41:28.507741179Z",
    "signature": "TransactionSignature",
    "slot": 1331
}
```

**字段说明:**
- `payer`: 支付者地址
- `order_pda`: 订单PDA地址
- `latest_price`: 最新价格
- `order_type`: 订单类型 (1: 做多/long, 2: 做空/short)
- `user`: 用户地址
- `lock_lp_start_price`: 锁定LP开始价格
- `lock_lp_end_price`: 锁定LP结束价格
- `lock_lp_sol_amount`: 锁定LP的SOL数量
- `lock_lp_token_amount`: 锁定LP的代币数量
- `start_time`: 开始时间 (Unix时间戳)
- `end_time`: 结束时间 (Unix时间戳)
- `margin_sol_amount`: 保证金SOL数量
- `borrow_amount`: 借贷数量
- `position_asset_amount`: 仓位资产数量
- `borrow_fee`: 借贷手续费

### 4. ForceLiquidate - 强制平仓事件

**事件类型**: `"ForceLiquidate"`

**数据格式:**
```json
{
    "event_type": "ForceLiquidate",
    "payer": "GKQXMqnr5kquB5KSngKt7gaY5yUmpp57PtDoTyqU",
    "mint_account": "4wuPEpuJXtoaJWEXmiSc4gN4nuGvcwx7dUGP8LHaMfvd",
    "order_pda": "DYSd91e5qcxRuAdu6f7mPE2Sx6xuGMqQRz4FENJ494Vq",
    "timestamp": "2025-09-25T12:41:28.507741179Z",
    "signature": "TransactionSignature",
    "slot": 1331
}
```

**字段说明:**
- `payer`: 执行强制平仓的地址
- `mint_account`: 代币mint地址
- `order_pda`: 被强制平仓的订单PDA地址

### 5. FullClose - 全部平仓事件

**事件类型**: `"FullClose"`

**数据格式:**
```json
{
    "event_type": "FullClose",
    "payer": "GKQXMqnr5kquB5KSngKt7gaY5yUmpp57PtDoTyqU",
    "user_sol_account": "GKQXMqnr5kquB5KSngKt7gaY5yUmpp57PtDoTyqU",
    "mint_account": "4wuPEpuJXtoaJWEXmiSc4gN4nuGvcwx7dUGP8LHaMfvd",
    "is_close_long": true,
    "final_token_amount": 128072830340859,
    "final_sol_amount": 3984795160,
    "user_close_profit": 1087998925,
    "latest_price": "279589934762348555452",
    "order_pda": "DYSd91e5qcxRuAdu6f7mPE2Sx6xuGMqQRz4FENJ494Vq",
    "timestamp": "2025-09-25T12:41:28.507741179Z",
    "signature": "TransactionSignature",
    "slot": 1332
}
```

**字段说明:**
- `payer`: 支付者地址
- `user_sol_account`: 用户SOL账户地址
- `is_close_long`: 是否为平多头仓位 (true: 平多, false: 平空)
- `final_token_amount`: 最终代币数量
- `final_sol_amount`: 最终SOL数量
- `user_close_profit`: 用户平仓盈利
- `latest_price`: 最新价格
- `order_pda`: 订单PDA地址

### 6. PartialClose - 部分平仓事件

**事件类型**: `"PartialClose"`

**数据格式:**
```json
{
    "event_type": "PartialClose",
    "payer": "GKQXMqnr5kquB5KSngKt7gaY5yUmpp57PtDoTyqU",
    "user_sol_account": "GKQXMqnr5kquB5KSngKt7gaY5yUmpp57PtDoTyqU",
    "mint_account": "4wuPEpuJXtoaJWEXmiSc4gN4nuGvcwx7dUGP8LHaMfvd",
    "is_close_long": true,
    "final_token_amount": 64036415170429,
    "final_sol_amount": 1992397580,
    "user_close_profit": 543999462,
    "latest_price": "279589934762348555452",
    "order_pda": "DYSd91e5qcxRuAdu6f7mPE2Sx6xuGMqQRz4FENJ494Vq",
    "order_type": 1,
    "mint": "4wuPEpuJXtoaJWEXmiSc4gN4nuGvcwx7dUGP8LHaMfvd",
    "user": "GKQXMqnr5kquB5KSngKt7gaY5yUmpp57PtDoTyqU",
    "lock_lp_start_price": "250000000000000000000",
    "lock_lp_end_price": "300000000000000000000",
    "lock_lp_sol_amount": 500000000,
    "lock_lp_token_amount": 250000000,
    "start_time": 1758805200,
    "end_time": 1758805800,
    "margin_sol_amount": 250000000,
    "borrow_amount": 750000000,
    "position_asset_amount": 1000000000,
    "borrow_fee": 125,
    "timestamp": "2025-09-25T12:41:28.507741179Z",
    "signature": "TransactionSignature",
    "slot": 1332
}
```

**字段说明:**
- 包含 `FullClose` 的所有字段
- 额外包含剩余订单的详细信息（与 `LongShort` 相同的字段）

### 7. MilestoneDiscount - 费率调整事件

**事件类型**: `"MilestoneDiscount"`

**数据格式:**
```json
{
    "event_type": "MilestoneDiscount",
    "payer": "GKQXMqnr5kquB5KSngKt7gaY5yUmpp57PtDoTyqU",
    "mint_account": "4wuPEpuJXtoaJWEXmiSc4gN4nuGvcwx7dUGP8LHaMfvd",
    "curve_account": "CurveAccountPubkey",
    "swap_fee": 50,
    "borrow_fee": 100,
    "fee_discount_flag": 1,
    "timestamp": "2025-09-25T12:41:28.507741179Z",
    "signature": "TransactionSignature",
    "slot": 1333
}
```

**字段说明:**
- `payer`: 触发费率调整的地址
- `curve_account`: 曲线账户地址
- `swap_fee`: 新的现货交易手续费
- `borrow_fee`: 新的保证金交易手续费
- `fee_discount_flag`: 新的手续费折扣标志

## 完整示例代码

```javascript
#!/usr/bin/env node

const { io } = require('socket.io-client');
const axios = require('axios');

const SERVER_URL = 'http://localhost:5051';
const INTERVAL = 's30';

// 获取最新的 mint 地址
async function getLatestMint() {
    const response = await axios.get(`${SERVER_URL}/api/mints`);
    return response.data.data.mints[0];
}

// 连接并订阅事件
async function subscribeToEvents() {
    const mint = await getLatestMint();
    
    const socket = io(`${SERVER_URL}/kline`, {
        transports: ['websocket', 'polling'],
        timeout: 20000,
        reconnection: true,
    });

    socket.on('connect', () => {
        console.log('连接成功');
        
        // 订阅K线（会同时收到事件数据）
        socket.emit('subscribe', {
            symbol: mint,
            interval: INTERVAL,
            subscription_id: `event_monitor_${Date.now()}`
        });
    });

    // 接收历史事件数据
    socket.on('history_event_data', (data) => {
        console.log(`收到${data.data.length}条历史事件`);
        data.data.forEach(event => {
            console.log(`事件类型: ${event.event_type}`);
        });
    });

    // 接收实时事件数据
    socket.on('event_data', (data) => {
        console.log(`实时事件: ${data.event_type}`);
        console.log('事件详情:', data.event_data);
    });

    return socket;
}

// 启动监听
subscribeToEvents().catch(console.error);
```

## 错误处理

```javascript
socket.on('error', (error) => {
    console.error('Socket错误:', error);
});

socket.on('connect_error', (error) => {
    console.error('连接错误:', error);
});

socket.on('disconnect', (reason) => {
    console.log('连接断开:', reason);
});
```

## 注意事项

1. **数据精度**: 价格字段使用字符串格式，具有28位小数精度
2. **时间戳**: 所有时间戳均为UTC时间格式
3. **数据量**: 历史事件最多返回300条，按slot倒序排列
4. **实时性**: 新事件会在产生后立即推送给所有订阅者
5. **K线依赖**: 必须先订阅K线才能接收事件数据
6. **连接管理**: 建议启用自动重连以确保数据连续性

## 数据存储格式

事件数据存储在 RocksDB 中，键值格式为：
```
tr:{mint_account}:{slot:010}:{event_type}:{signature}
```

其中 `event_type` 的映射关系：
- `tc`: TokenCreated
- `bs`: BuySell  
- `ls`: LongShort
- `fl`: ForceLiquidate
- `fc`: FullClose
- `pc`: PartialClose
- `md`: MilestoneDiscount