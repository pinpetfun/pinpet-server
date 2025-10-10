# Socket.IO K线实时推送协议文档

## 概述

本项目基于 SocketIoxide 0.17 实现了 Socket.IO 实时 K线数据推送服务，为客户端提供高效的加密货币价格数据订阅功能。

## 服务端点

- **WebSocket 地址**: `ws://[SERVER_HOST]:[SERVER_PORT]/kline`
- **默认端口**: 5051
- **协议**: Socket.IO v4 兼容
- **传输方式**: WebSocket, Polling
- **命名空间**: `/kline`

## 连接配置

### 服务器配置参数

| 参数 | 默认值 | 描述 |
|------|--------|------|
| `connection_timeout` | 60秒 | 连接超时时间 |
| `max_subscriptions_per_client` | 100 | 每客户端最大订阅数 |
| `history_data_limit` | 100 | 历史数据默认条数 |
| `ping_interval` | 25秒 | 心跳间隔 |
| `ping_timeout` | 60秒 | 心跳超时 |

### 客户端连接示例

```javascript
const { io } = require('socket.io-client');

const socket = io('ws://localhost:5051/kline', {
    transports: ['websocket', 'polling'],
    timeout: 20000,
    reconnection: true,
    reconnectionAttempts: 5,
    reconnectionDelay: 1000,
});
```

## 支持的时间间隔

- `s1`: 1秒K线
- `s30`: 30秒K线  
- `m5`: 5分钟K线

## 协议事件

### 1. 连接成功 (connection_success)

**触发时机**: 客户端成功连接到服务器后  
**方向**: 服务器 → 客户端

```json
{
    "client_id": "socket_id_string",
    "server_time": 1694567890,
    "supported_symbols": [],
    "supported_intervals": ["s1", "s30", "m5"]
}
```

### 2. 订阅K线数据 (subscribe)

**触发时机**: 客户端请求订阅特定代币的K线数据  
**方向**: 客户端 → 服务器

#### 请求格式

```json
{
    "symbol": "DTyWzeXUXXaYqJAbqP3J2wS4WJHrBz9NauNi63hBdjQP",
    "interval": "s1",
    "subscription_id": "optional_client_id"
}
```

#### 字段说明

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `symbol` | String | ✅ | 代币地址 (32-44字符的Solana地址) |
| `interval` | String | ✅ | 时间间隔 (s1/s30/m5) |
| `subscription_id` | String | ❌ | 客户端订阅ID |

### 3. 订阅确认 (subscription_confirmed)

**触发时机**: 服务器确认订阅成功  
**方向**: 服务器 → 客户端

```json
{
    "symbol": "DTyWzeXUXXaYqJAbqP3J2wS4WJHrBz9NauNi63hBdjQP",
    "interval": "s1",
    "subscription_id": "optional_client_id",
    "success": true,
    "message": "订阅成功"
}
```

### 4. 历史数据响应 (history_data)

**触发时机**: 订阅成功后自动推送，或响应历史数据请求  
**方向**: 服务器 → 客户端

```json
{
    "symbol": "DTyWzeXUXXaYqJAbqP3J2wS4WJHrBz9NauNi63hBdjQP",
    "interval": "s1",
    "data": [
        {
            "time": 1694567890,
            "open": 1.23,
            "high": 1.45,
            "low": 1.10,
            "close": 1.35,
            "volume": 0.0,
            "is_final": false,
            "update_type": "final",
            "update_count": 5
        }
    ],
    "has_more": true,
    "total_count": 150
}
```

### 5. 实时K线数据 (kline_data)

**触发时机**: 有新的价格更新时实时推送  
**方向**: 服务器 → 客户端

```json
{
    "symbol": "DTyWzeXUXXaYqJAbqP3J2wS4WJHrBz9NauNi63hBdjQP",
    "interval": "s1",
    "subscription_id": null,
    "data": {
        "time": 1694567890,
        "open": 1.23,
        "high": 1.45,
        "low": 1.10,
        "close": 1.35,
        "volume": 0.0,
        "is_final": false,
        "update_type": "realtime",
        "update_count": 3
    },
    "timestamp": 1694567890123
}
```

#### K线数据字段说明

| 字段 | 类型 | 描述 |
|------|------|------|
| `time` | Number | Unix时间戳 (秒) |
| `open` | Number | 开盘价 |
| `high` | Number | 最高价 |
| `low` | Number | 最低价 |
| `close` | Number | 收盘价 (当前价格) |
| `volume` | Number | 成交量 (当前恒为0) |
| `is_final` | Boolean | 是否为最终K线 |
| `update_type` | String | 更新类型: "realtime" \| "final" |
| `update_count` | Number | 更新次数 |

### 6. 获取历史数据 (history)

**触发时机**: 客户端主动请求历史数据  
**方向**: 客户端 → 服务器

```json
{
    "symbol": "DTyWzeXUXXaYqJAbqP3J2wS4WJHrBz9NauNi63hBdjQP",
    "interval": "s1",
    "limit": 10,
    "from": 1694567890
}
```

### 7. 取消订阅 (unsubscribe)

**触发时机**: 客户端取消特定订阅  
**方向**: 客户端 → 服务器

```json
{
    "symbol": "DTyWzeXUXXaYqJAbqP3J2wS4WJHrBz9NauNi63hBdjQP",
    "interval": "s1",
    "subscription_id": "optional_client_id"
}
```

### 8. 取消订阅确认 (unsubscribe_confirmed)

**触发时机**: 服务器确认取消订阅成功  
**方向**: 服务器 → 客户端

```json
{
    "symbol": "DTyWzeXUXXaYqJAbqP3J2wS4WJHrBz9NauNi63hBdjQP",
    "interval": "s1",
    "subscription_id": "optional_client_id",
    "success": true
}
```

### 9. 错误消息 (error)

**触发时机**: 发生错误时  
**方向**: 服务器 → 客户端

```json
{
    "code": 1001,
    "message": "Invalid interval: invalid, must be one of: s1, s30, m5"
}
```

#### 错误代码

| 错误代码 | 描述 |
|----------|------|
| 1001 | 无效的订阅参数 |
| 1002 | 订阅数量超限 |
| 1003 | 历史数据查询失败 |

## 服务器内部实现

### 房间管理

服务器使用房间 (Room) 系统管理订阅:
- 房间格式: `kline:{mint_account}:{interval}`
- 示例: `kline:DTyWzeXUXXaYqJAbqP3J2wS4WJHrBz9NauNi63hBdjQP:s1`

### 订阅管理

```rust
pub struct SubscriptionManager {
    // 连接映射: SocketId -> 客户端信息
    pub connections: HashMap<String, ClientConnection>,
    
    // 订阅索引: mint_account -> interval -> SocketId集合
    pub mint_subscribers: HashMap<String, HashMap<String, HashSet<String>>>,
    
    // 反向索引: SocketId -> 订阅键集合
    pub client_subscriptions: HashMap<String, HashSet<String>>,
}
```

### 事件处理流程

1. **连接建立**: 客户端连接 → 注册到连接管理器 → 发送欢迎消息
2. **订阅处理**: 验证请求 → 添加到房间 → 推送历史数据 → 确认订阅
3. **实时推送**: 区块链事件 → K线计算 → 房间广播 → 客户端接收
4. **连接清理**: 定时任务清理超时连接 → 自动取消相关订阅

## 客户端集成示例

### 基础连接和订阅

```javascript
const { io } = require('socket.io-client');

const socket = io('ws://localhost:5051/kline');

socket.on('connect', () => {
    console.log('连接成功');
    
    // 订阅某个代币的1秒K线
    socket.emit('subscribe', {
        symbol: 'DTyWzeXUXXaYqJAbqP3J2wS4WJHrBz9NauNi63hBdjQP',
        interval: 's1',
        subscription_id: 'my_subscription_1'
    });
});

socket.on('kline_data', (data) => {
    console.log('收到K线更新:', {
        symbol: data.symbol,
        price: data.data.close,
        time: new Date(data.data.time * 1000)
    });
});

socket.on('error', (error) => {
    console.error('发生错误:', error);
});
```

### 多订阅管理

```javascript
const subscriptions = new Map();

function subscribeSymbol(symbol, interval) {
    const subscriptionId = `${symbol}_${interval}_${Date.now()}`;
    
    socket.emit('subscribe', {
        symbol,
        interval,
        subscription_id: subscriptionId
    });
    
    subscriptions.set(subscriptionId, { symbol, interval });
    return subscriptionId;
}

function unsubscribeSymbol(subscriptionId) {
    const sub = subscriptions.get(subscriptionId);
    if (sub) {
        socket.emit('unsubscribe', {
            symbol: sub.symbol,
            interval: sub.interval,
            subscription_id: subscriptionId
        });
        subscriptions.delete(subscriptionId);
    }
}
```

## 性能特性

### 连接限制
- 每个客户端最多100个订阅
- 连接超时60秒自动清理
- 支持自动重连

### 数据推送
- 基于房间的高效广播
- 增量更新减少带宽
- 实时性: 毫秒级延迟

### 监控统计
- 连接数量统计
- 消息发送计数
- 性能指标监控

## 故障处理

### 客户端重连策略

```javascript
const socket = io('ws://localhost:5051/kline', {
    reconnection: true,
    reconnectionAttempts: 10,
    reconnectionDelay: 2000,
    reconnectionDelayMax: 10000,
});

socket.on('reconnect', () => {
    console.log('重连成功，恢复订阅...');
    // 重新订阅之前的symbol
    restoreSubscriptions();
});
```

### 服务器端清理

- 定期清理超时连接 (30秒间隔)
- 自动移除无效订阅
- 内存使用优化

## 测试工具

项目提供了多个测试脚本:

1. **基础功能测试**: `test_kline_websocket.js`
2. **自动监听测试**: `test/test_auto_kline.js`
3. **压力测试**: `stress_test_kline.js`

运行测试:
```bash
node test_kline_websocket.js
node test/test_auto_kline.js
```

## 相关配置

在 `config/default.toml` 中配置K线服务:

```toml
[kline]
enable_kline_service = true
connection_timeout_secs = 60
max_subscriptions_per_client = 100
history_data_limit = 100
ping_interval_secs = 25
ping_timeout_secs = 60
```