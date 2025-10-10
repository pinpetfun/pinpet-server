# K线行情 WebSocket 接口说明

## 概述

K线行情服务提供实时的加密货币价格数据推送，支持多种时间间隔的K线数据订阅。服务基于 Socket.IO 协议，使用 `/kline` 命名空间。

## 连接信息

- **Socket.IO 服务器**: `ws://0.0.0.0:5051`
- **命名空间**: `/kline`
- **完整连接URL**: `ws://0.0.0.0:5051/kline`
- **协议**: Socket.IO v4+
- **支持的传输方式**: WebSocket, Polling

> **说明**: Socket.IO 会自动处理底层的 `/socket.io/` 传输路径，客户端只需连接到对应的命名空间即可。

## 支持的时间间隔

- `s1` - 1秒K线
- `s30` - 30秒K线
- `m5` - 5分钟K线

## JavaScript 客户端集成

### 1. 安装依赖

```bash
npm install socket.io-client
```

### 2. 基础连接示例

```javascript
const { io } = require('socket.io-client');

// 连接到K线服务的 /kline 命名空间
const socket = io('http://localhost:5051/kline', {
    transports: ['websocket', 'polling'],
    timeout: 20000,
    reconnection: true,
    reconnectionAttempts: 5,
    reconnectionDelay: 1000,
});

// 监听连接成功
socket.on('connect', () => {
    console.log('✅ 连接成功');
    console.log('Socket ID:', socket.id);
});

// 监听连接断开
socket.on('disconnect', (reason) => {
    console.log('❌ 连接断开:', reason);
});

// 监听连接错误
socket.on('connect_error', (error) => {
    console.log('💥 连接错误:', error.message);
});
```

### 3. 订阅K线数据

```javascript
// 订阅请求
socket.emit('subscribe', {
    symbol: 'JBMmrp6jhksqnxDBskkmVvWHhJLaPBjgiMHEroJbUTBZ', // 代币地址
    interval: 's1',                                         // 时间间隔
    subscription_id: 'my_subscription_123'                  // 可选：订阅ID
});

// 监听订阅确认
socket.on('subscription_confirmed', (data) => {
    console.log('✅ 订阅确认:', data);
    // 输出示例:
    // {
    //   "symbol": "JBMmrp6jhksqnxDBskkmVvWHhJLaPBjgiMHEroJbUTBZ",
    //   "interval": "s1",
    //   "subscription_id": "my_subscription_123",
    //   "success": true,
    //   "message": "订阅成功"
    // }
});
```

### 4. 接收实时K线数据

```javascript
// 监听实时K线数据推送
socket.on('kline_data', (data) => {
    console.log('📊 实时K线更新:', data);
    // 数据结构:
    // {
    //   "symbol": "JBMmrp6jhksqnxDBskkmVvWHhJLaPBjgiMHEroJbUTBZ",
    //   "interval": "s1",
    //   "subscription_id": null,
    //   "data": {
    //     "time": 1640995200,           // Unix时间戳（秒）
    //     "open": 1.23,                 // 开盘价
    //     "high": 1.45,                 // 最高价
    //     "low": 1.10,                  // 最低价
    //     "close": 1.35,                // 收盘价（当前价格）
    //     "volume": 0.0,                // 成交量（当前为0）
    //     "is_final": false,            // 是否为最终K线
    //     "update_type": "realtime",    // 更新类型: "realtime" | "final"
    //     "update_count": 5             // 更新次数
    //   },
    //   "timestamp": 1640995200123      // 推送时间戳（毫秒）
    // }
});
```

### 5. 获取历史数据

```javascript
// 请求历史数据
socket.emit('history', {
    symbol: 'JBMmrp6jhksqnxDBskkmVvWHhJLaPBjgiMHEroJbUTBZ',
    interval: 's1',
    limit: 100,        // 可选：数据条数，默认100
    from: 1640995200   // 可选：开始时间戳（秒）
});

// 监听历史数据响应
socket.on('history_data', (data) => {
    console.log('📈 历史数据:', data);
    // 数据结构:
    // {
    //   "symbol": "JBMmrp6jhksqnxDBskkmVvWHhJLaPBjgiMHEroJbUTBZ",
    //   "interval": "s1",
    //   "data": [ /* K线数据数组 */ ],
    //   "has_more": true,             // 是否还有更多数据
    //   "total_count": 1000          // 总数据条数
    // }
});
```

### 6. 取消订阅

```javascript
// 取消订阅
socket.emit('unsubscribe', {
    symbol: 'JBMmrp6jhksqnxDBskkmVvWHhJLaPBjgiMHEroJbUTBZ',
    interval: 's1',
    subscription_id: 'my_subscription_123'  // 可选
});

// 监听取消订阅确认
socket.on('unsubscribe_confirmed', (data) => {
    console.log('🚫 取消订阅确认:', data);
});
```

### 7. 错误处理

```javascript
// 监听服务器错误
socket.on('error', (error) => {
    console.log('❌ 服务器错误:', error);
    // 错误结构:
    // {
    //   "code": 1001,
    //   "message": "Invalid interval: invalid, must be one of: s1, s30, m5"
    // }
});
```

## 完整示例代码

```javascript
const { io } = require('socket.io-client');

const SERVER_URL = 'http://localhost:5051';
const TEST_MINT = 'JBMmrp6jhksqnxDBskkmVvWHhJLaPBjgiMHEroJbUTBZ';

// 创建连接
const socket = io(`${SERVER_URL}/kline`, {
    transports: ['websocket', 'polling'],
    timeout: 20000,
    reconnection: true,
    reconnectionAttempts: 5,
    reconnectionDelay: 1000,
});

// 连接事件
socket.on('connect', () => {
    console.log('✅ 连接成功, Socket ID:', socket.id);
    
    // 订阅多个间隔的K线数据
    ['s1', 's30', 'm5'].forEach(interval => {
        socket.emit('subscribe', {
            symbol: TEST_MINT,
            interval: interval,
            subscription_id: `sub_${interval}_${Date.now()}`
        });
    });
    
    // 获取历史数据
    setTimeout(() => {
        socket.emit('history', {
            symbol: TEST_MINT,
            interval: 's1',
            limit: 10
        });
    }, 1000);
});

socket.on('disconnect', (reason) => {
    console.log('❌ 连接断开:', reason);
});

socket.on('connect_error', (error) => {
    console.log('💥 连接错误:', error.message);
});

// 数据事件
socket.on('connection_success', (data) => {
    console.log('🎉 连接成功消息:', data);
});

socket.on('subscription_confirmed', (data) => {
    console.log('✅ 订阅确认:', data);
});

socket.on('history_data', (data) => {
    console.log(`📈 历史数据 (${data.symbol}@${data.interval}):`, {
        dataPoints: data.data.length,
        hasMore: data.has_more,
        totalCount: data.total_count
    });
});

socket.on('kline_data', (data) => {
    console.log(`📊 实时K线 (${data.symbol}@${data.interval}):`, {
        time: new Date(data.data.time * 1000).toISOString(),
        price: data.data.close,
        updateType: data.data.update_type,
        updateCount: data.data.update_count
    });
});

socket.on('error', (error) => {
    console.log('❌ 错误:', error);
});

// 优雅退出
process.on('SIGINT', () => {
    console.log('\\n👋 正在断开连接...');
    socket.disconnect();
    process.exit(0);
});
```

## 测试代码使用

### 1. 运行完整测试

```bash
# 使用项目提供的测试脚本
./run_kline_test.sh
```

该脚本会：
- 启动服务器
- 运行 WebSocket 连接测试
- 执行订阅/取消订阅流程
- 进行压力测试

### 2. 单独运行 WebSocket 测试

```bash
# 确保服务器正在运行
cargo run

# 在另一个终端运行测试
node test_kline_websocket.js
```

### 3. 运行压力测试

```bash
node stress_test_kline.js
```

压力测试会创建多个并发客户端，每个客户端订阅多个时间间隔的数据，用于测试服务器性能。

## 错误代码说明

| 错误代码 | 说明 |
|---------|------|
| 1001 | 无效的时间间隔参数 |
| 1002 | 订阅数量超过限制 |
| 1003 | 获取历史数据失败 |

## 性能特性

- **连接限制**: 每个客户端最多100个订阅
- **数据限制**: 单次历史数据查询最多返回100条记录
- **重连机制**: 支持自动重连，最多5次尝试
- **心跳检测**: 25秒心跳间隔，60秒超时
- **传输优化**: 优先使用WebSocket，降级到轮询

## 注意事项

1. **代币地址格式**: 必须是有效的Solana地址格式（32-44字符）
2. **时间间隔**: 仅支持 `s1`, `s30`, `m5` 三种间隔
3. **连接稳定性**: 建议启用自动重连机制
4. **数据实时性**: 实时数据推送延迟通常在100ms以内
5. **资源管理**: 不需要的订阅应及时取消以节省资源