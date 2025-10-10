# WebSocket 命名空间问题解决方案

## 问题描述

在使用 Rust 的 socketioxide 库构建 Socket.IO 服务器时，客户端能够成功连接、订阅，但无法接收到服务器发送的实时数据（`kline_data` 事件）。

### 症状表现

1. **客户端连接正常**：能够收到 `connection_success` 事件
2. **订阅成功**：能够收到 `subscription_confirmed` 事件  
3. **历史数据正常**：能够收到 `history_data` 事件
4. **实时数据丢失**：无法收到 `kline_data` 事件
5. **服务器统计显示正常**：`kline_data_sent` 计数正在增长

### 调试过程

通过以下方式进行排查：
- 客户端使用 `socket.onAny()` 捕获所有事件
- 服务器端添加详细的发送日志
- 检查订阅状态和房间成员
- 尝试直接向 socket 发送测试消息

## 根本原因

**socketioxide 库的命名空间处理机制与标准 Socket.IO 不同**：

1. **命名空间隔离**：在 socketioxide 中，不同命名空间之间完全隔离
2. **房间作用域**：房间只在特定命名空间内有效
3. **发送方式错误**：使用全局 `socketio.to(room).emit()` 无法正确发送到命名空间内的房间

### 错误的代码模式

```rust
// ❌ 错误：发送到全局作用域，无法到达命名空间内的客户端
self.socketio.to(room_name).emit("kline_data", &message).await
```

### 正确的代码模式

```rust
// ✅ 正确：明确指定命名空间后发送到房间
self.socketio.of("/kline")
    .ok_or_else(|| anyhow::anyhow!("Namespace /kline not found"))?
    .to(room_name)
    .emit("kline_data", &message)
    .await
```

## 解决方案

### 1. 修复房间广播代码

在 `src/services/kline_socket.rs` 的 `broadcast_kline_update` 方法中：

```rust
// 修改前
let result = self.socketio.to(room_name.clone()).emit("kline_data", &update_message).await;

// 修改后  
let result = self.socketio.of("/kline")
    .ok_or_else(|| anyhow::anyhow!("Namespace /kline not found"))?
    .to(room_name.clone())
    .emit("kline_data", &update_message)
    .await;
```

### 2. 确保命名空间正确创建

在 `setup_socket_handlers` 方法中确认命名空间设置：

```rust
// 设置默认命名空间
self.socketio.ns("/", |_socket: SocketRef| {
    // 默认命名空间处理
});

// 设置 K线 命名空间  
self.socketio.ns("/kline", move |socket: SocketRef| {
    // K线相关事件处理
});
```

### 3. 客户端连接验证

确保客户端正确连接到指定命名空间：

```javascript
// 客户端连接到 /kline 命名空间
const socket = io(`${SERVER_URL}/kline`, {
    transports: ['websocket', 'polling'],
    timeout: 20000,
    reconnection: true,
});
```

## 技术要点

### socketioxide vs Socket.IO

| 特性 | Socket.IO (Node.js) | socketioxide (Rust) |
|-----|-------------------|-------------------|
| 默认命名空间 | 自动创建 `/` | 需要手动创建 |
| 跨命名空间发送 | 支持全局发送 | 必须指定命名空间 |
| 房间作用域 | 全局房间 | 命名空间内房间 |

### 调试技巧

1. **使用命名空间特定的发送方式**
2. **添加详细的发送日志以追踪消息流**
3. **客户端使用 `onAny()` 捕获所有事件**
4. **检查服务器订阅统计 API**

## 最佳实践

### 1. 明确命名空间发送

```rust
// 总是明确指定目标命名空间
if let Some(ns) = self.socketio.of("/kline") {
    ns.to(room_name).emit("event_name", &data).await?;
}
```

### 2. 错误处理

```rust
// 添加适当的错误处理
let result = self.socketio.of("/kline")
    .ok_or_else(|| anyhow::anyhow!("Namespace /kline not found"))?
    .to(room_name)
    .emit("kline_data", &message)
    .await;

match result {
    Ok(_) => info!("Successfully sent to room: {}", room_name),
    Err(e) => warn!("Failed to send to room {}: {}", room_name, e),
}
```

### 3. 监控和调试

```rust
// 添加发送前的状态检查
{
    let manager = self.subscriptions.read().await;
    let subscribers = manager.get_subscribers(mint_account, interval);
    info!("Room {} has {} subscribers: {:?}", room_name, subscribers.len(), subscribers);
}
```

## 总结

这个问题的核心在于理解 socketioxide 库的命名空间隔离机制。与标准的 Socket.IO 实现不同，socketioxide 要求在发送消息到房间时必须明确指定命名空间。

通过明确使用 `socketio.of(namespace).to(room).emit()` 的模式，可以确保消息正确发送到指定命名空间内的房间订阅者。

这个修复确保了：
- 实时 K线数据能够正确推送给客户端
- 维持了命名空间的正确隔离
- 提供了清晰的错误处理机制

## 相关文件

- `src/services/kline_socket.rs` - 主要修复文件
- `test_auto_kline.js` - 客户端测试脚本  
- `src/main.rs` - 服务初始化配置