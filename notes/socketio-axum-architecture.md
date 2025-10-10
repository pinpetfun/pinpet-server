# Axum + SocketIoxide 高性能 WebSocket 架构设计

## 概述

基于 `notes/socketio-kline-protocol.md` 的协议设计，本文档详细分析了使用 Axum + socketioxide 实现高性能 Socket.IO 服务的架构方案。该方案充分利用了项目现有的 Solana 事件监听、RocksDB 存储和 Axum HTTP 服务基础设施。

## 项目现有架构分析

### 当前技术栈 
- **Web框架**: Axum 0.7 + Tower 中间件
- **异步运行时**: Tokio (full features)
- **数据存储**: RocksDB 键值存储
- **区块链集成**: Solana SDK + tokio-tungstenite (已有 WebSocket 支持)
- **配置管理**: config crate + TOML
- **日志系统**: tracing + tracing-subscriber

### 现有服务结构
```rust
AppState {
    event_service: Arc<RwLock<EventService>>,  // 事件处理服务
    event_storage: Arc<EventStorage>           // RocksDB 存储服务
}
```

### 数据流向
```
Solana 区块链 → EventListenerManager → StatsEventHandler → RocksDB
                                                      ↓
                                              统计数据更新
```

## Socket.IO 架构设计

### 依赖添加

```toml
# Cargo.toml 新增依赖
socketioxide = { version = "0.15", features = ["tracing"] }
dashmap = "6.1"              # 高性能并发 HashMap
uuid = { version = "1.11", features = ["v4", "fast-rng"] }
tokio-stream = "0.1"         # 流处理工具
futures-util = "0.3"        # 已有，扩展使用
```

### 核心架构组件

#### 1. WebSocket 连接管理器

```rust
// src/websocket/connection_manager.rs
use dashmap::{DashMap, DashSet};
use socketioxide::{extract::{SocketRef, Data}, SocketIo};
use std::sync::Arc;

pub struct WSConnectionManager {
    // 连接池：client_id -> SocketRef
    connections: DashMap<String, SocketRef>,
    
    // 订阅关系：topic -> Set<client_id>
    subscriptions: DashMap<String, DashSet<String>>,
    
    // 客户端订阅：client_id -> Set<topic>
    client_subscriptions: DashMap<String, DashSet<String>>,
    
    // 连接统计
    stats: Arc<RwLock<ConnectionStats>>,
}

#[derive(Debug, Clone)]
pub struct ConnectionStats {
    pub total_connections: u64,
    pub active_connections: u64,
    pub total_messages_sent: u64,
    pub total_subscriptions: u64,
}
```

#### 2. K线数据管理器

```rust
// src/websocket/kline_manager.rs
use std::collections::BTreeMap;

pub struct KlineManager {
    // 当前活跃的 K线数据：symbol_interval -> KlineData
    active_klines: DashMap<String, KlineData>,
    
    // 价格缓存：symbol -> LatestPrice
    price_cache: DashMap<String, LatestPrice>,
    
    // 事件存储引用
    event_storage: Arc<EventStorage>,
    
    // 广播器
    broadcaster: Arc<WSBroadcaster>,
}

#[derive(Debug, Clone, Serialize)]
pub struct KlineData {
    pub time: u64,
    pub open: f64,
    pub high: f64,
    pub low: f64,
    pub close: f64,
    pub volume: f64,
    pub is_final: bool,
    pub update_count: u32,  // 防止重复更新
}
```

#### 3. 高性能广播器

```rust
// src/websocket/broadcaster.rs
pub struct WSBroadcaster {
    connection_manager: Arc<WSConnectionManager>,
    
    // 消息队列：避免阻塞主线程
    message_queue: Arc<tokio::sync::mpsc::UnboundedSender<BroadcastMessage>>,
    
    // 批量发送控制
    batch_size: usize,
    batch_timeout: Duration,
}

#[derive(Debug, Clone)]
pub struct BroadcastMessage {
    pub topic: String,
    pub event: String,
    pub data: serde_json::Value,
    pub timestamp: i64,
}

impl WSBroadcaster {
    // 高性能广播实现
    pub async fn broadcast_to_topic(&self, topic: &str, event: &str, data: serde_json::Value) {
        // 获取订阅者列表
        let subscribers = self.connection_manager.get_topic_subscribers(topic);
        
        if subscribers.is_empty() {
            return;
        }
        
        // 并发发送，不等待结果
        let futures: Vec<_> = subscribers.into_iter()
            .filter_map(|client_id| {
                self.connection_manager.get_connection(&client_id)
            })
            .map(|socket| {
                let event = event.to_string();
                let data = data.clone();
                async move {
                    let _ = socket.emit(event, &data);
                }
            })
            .collect();
        
        // 使用 join_all 并发发送
        futures::future::join_all(futures).await;
    }
    
    // 批量发送优化
    pub async fn start_batch_processor(&self) {
        let mut batch = Vec::with_capacity(self.batch_size);
        let mut interval = tokio::time::interval(self.batch_timeout);
        
        loop {
            tokio::select! {
                msg = self.message_queue.recv() => {
                    if let Some(msg) = msg {
                        batch.push(msg);
                        
                        if batch.len() >= self.batch_size {
                            self.process_batch(&mut batch).await;
                        }
                    }
                }
                _ = interval.tick() => {
                    if !batch.is_empty() {
                        self.process_batch(&mut batch).await;
                    }
                }
            }
        }
    }
}
```

### 与现有系统集成

#### 1. 扩展 AppState

```rust
// src/handlers.rs 修改
pub struct AppState {
    pub event_service: Arc<tokio::sync::RwLock<EventService>>,
    pub event_storage: Arc<EventStorage>,
    
    // 新增 WebSocket 组件
    pub ws_connection_manager: Arc<WSConnectionManager>,
    pub kline_manager: Arc<KlineManager>,
    pub ws_broadcaster: Arc<WSBroadcaster>,
}
```

#### 2. 事件处理器增强

```rust
// src/services/event_service.rs 扩展
impl StatsEventHandler {
    // 新增 WebSocket 广播功能
    pub fn with_websocket_broadcaster(
        event_storage: Arc<EventStorage>,
        ws_broadcaster: Arc<WSBroadcaster>,
        kline_manager: Arc<KlineManager>,
    ) -> Self {
        Self {
            stats: Arc::new(RwLock::new(EventStats::default())),
            last_event_time: Arc::new(RwLock::new(None)),
            event_storage,
            ws_broadcaster: Some(ws_broadcaster),
            kline_manager: Some(kline_manager),
        }
    }
}

#[async_trait::async_trait]
impl EventHandler for StatsEventHandler {
    async fn handle_event(&self, event: SpinPetEvent) -> anyhow::Result<()> {
        // 原有逻辑：存储 + 统计
        self.store_and_update_stats(event.clone()).await?;
        
        // 新增：实时 WebSocket 推送
        if let Some(broadcaster) = &self.ws_broadcaster {
            self.broadcast_event(event, broadcaster).await?;
        }
        
        Ok(())
    }
    
    async fn broadcast_event(
        &self, 
        event: SpinPetEvent, 
        broadcaster: &WSBroadcaster
    ) -> anyhow::Result<()> {
        match event {
            SpinPetEvent::BuySell(buy_sell) => {
                // 更新价格数据
                if let Some(kline_manager) = &self.kline_manager {
                    kline_manager.update_price(&buy_sell.mint, buy_sell.price).await;
                }
                
                // 广播价格更新
                broadcaster.broadcast_to_topic(
                    &format!("price:{}", buy_sell.mint),
                    "price_update",
                    serde_json::json!({
                        "symbol": buy_sell.mint,
                        "price": buy_sell.price,
                        "timestamp": buy_sell.timestamp,
                        "side": buy_sell.side
                    })
                ).await;
            }
            SpinPetEvent::TokenCreated(token) => {
                // 广播新币创建
                broadcaster.broadcast_to_topic(
                    "token_events",
                    "token_created",
                    serde_json::to_value(token)?
                ).await;
            }
            // ... 其他事件类型
        }
        Ok(())
    }
}
```

#### 3. 路由集成

```rust
// src/routes.rs 修改
use socketioxide::SocketIo;

pub fn create_router(config: &Config, app_state: Arc<AppState>) -> Router {
    // 创建 Socket.IO 服务
    let (layer, io) = SocketIo::new_layer();
    
    // 注册 Socket.IO 事件处理器
    io.ns("/ws/kline", move |socket: SocketRef| {
        let app_state = app_state.clone();
        async move {
            setup_kline_handlers(socket, app_state).await;
        }
    });
    
    let app = Router::new()
        // 现有 API 路由
        .route("/api/time", get(handlers::get_time))
        .route("/api/events/status", get(handlers::get_event_status))
        // ... 其他路由
        
        // Socket.IO 路由 - 最重要的变化
        .layer(layer)
        
        // 应用状态
        .with_state(app_state)
        
        // 现有中间件
        .layer(create_cors_layer(&config.cors.allow_origins))
        .layer(TraceLayer::new_for_http());
    
    app
}

// 请求/响应结构体定义
#[derive(Deserialize)]
struct SubscribeRequest {
    symbol: String,
    interval: String,
}

#[derive(Deserialize)]
struct UnsubscribeRequest {
    symbol: String,
    interval: String,
}

#[derive(Deserialize)]
struct HistoryRequest {
    symbol: String,
    interval: String,
    limit: Option<u32>,
}

// Socket.IO 事件处理器
async fn setup_kline_handlers(socket: SocketRef, app_state: Arc<AppState>) {
    let client_id = uuid::Uuid::new_v4().to_string();
    
    // 注册连接
    app_state.ws_connection_manager.add_connection(client_id.clone(), socket.clone());
    
    // 发送连接成功消息
    let _ = socket.emit("connection_success", &serde_json::json!({
        "clientId": client_id,
        "serverTime": chrono::Utc::now().timestamp(),
        "supportedSymbols": ["BTCUSDT", "ETHUSDT", "SOLUSDT"],
        "supportedIntervals": ["s1", "m1", "m5"]
    }));
    
    // 订阅事件处理器
    socket.on("subscribe_kline", {
        let app_state = app_state.clone();
        let client_id = client_id.clone();
        move |socket: SocketRef, Data::<SubscribeRequest>(data)| {
            let app_state = app_state.clone();
            let client_id = client_id.clone();
            async move {
                handle_subscribe(socket, data.0, app_state, client_id).await;
            }
        }
    });
    
    // 取消订阅处理器
    socket.on("unsubscribe_kline", {
        let app_state = app_state.clone();
        let client_id = client_id.clone();
        move |socket: SocketRef, Data::<UnsubscribeRequest>(data)| {
            let app_state = app_state.clone();
            let client_id = client_id.clone();
            async move {
                handle_unsubscribe(socket, data.0, app_state, client_id).await;
            }
        }
    });
    
    // 历史数据请求处理器
    socket.on("get_history", {
        let app_state = app_state.clone();
        move |socket: SocketRef, Data::<HistoryRequest>(data)| {
            let app_state = app_state.clone();
            async move {
                handle_get_history(socket, data.0, app_state).await;
            }
        }
    });
    
    // 断开连接处理
    socket.on_disconnect({
        let app_state = app_state.clone();
        let client_id = client_id.clone();
        move |socket: SocketRef, reason| {
            let app_state = app_state.clone();
            let client_id = client_id.clone();
            async move {
                tracing::info!("Client {} disconnected: {:?}", client_id, reason);
                app_state.ws_connection_manager.remove_connection(&client_id);
            }
        }
    });
}
```

## 性能优化策略

### 1. 连接池管理优化

```rust
impl WSConnectionManager {
    // 连接健康检查
    pub async fn start_health_checker(&self) {
        let mut interval = tokio::time::interval(Duration::from_secs(30));
        
        loop {
            interval.tick().await;
            self.cleanup_stale_connections().await;
        }
    }
    
    async fn cleanup_stale_connections(&self) {
        self.connections.retain(|_, socket| {
            socket.connected()
        });
    }
    
    // 限制单客户端订阅数
    pub fn can_subscribe(&self, client_id: &str) -> bool {
        if let Some(subs) = self.client_subscriptions.get(client_id) {
            subs.len() < 50  // 限制最多 50 个订阅
        } else {
            true
        }
    }
}
```

### 2. 内存优化

```rust
// K线数据缓存策略
impl KlineManager {
    // LRU 缓存清理
    pub async fn start_cache_cleaner(&self) {
        let mut interval = tokio::time::interval(Duration::from_secs(300));
        
        loop {
            interval.tick().await;
            self.cleanup_old_klines().await;
        }
    }
    
    async fn cleanup_old_klines(&self) {
        let cutoff = chrono::Utc::now().timestamp() - 3600; // 保留 1 小时
        
        self.active_klines.retain(|_key, kline| {
            kline.time > cutoff
        });
    }
}
```

### 3. 消息批处理

```rust
// 批量消息处理
impl WSBroadcaster {
    async fn process_batch(&self, batch: &mut Vec<BroadcastMessage>) {
        // 按 topic 分组
        let mut grouped = HashMap::new();
        
        for msg in batch.drain(..) {
            grouped.entry(msg.topic.clone())
                   .or_insert_with(Vec::new)
                   .push(msg);
        }
        
        // 并发处理每个 topic
        let futures: Vec<_> = grouped.into_iter()
            .map(|(topic, messages)| {
                self.broadcast_batch_to_topic(topic, messages)
            })
            .collect();
        
        futures::future::join_all(futures).await;
    }
}
```

## 监控和调试

### 1. 性能指标

```rust
#[derive(Debug, Serialize)]
pub struct WSMetrics {
    pub connections: ConnectionStats,
    pub messages_per_second: f64,
    pub average_latency_ms: f64,
    pub error_rate: f64,
    pub memory_usage_mb: f64,
}

// 在 /api/ws/metrics 端点暴露
pub async fn get_ws_metrics(
    State(state): State<Arc<AppState>>
) -> ResponseJson<ApiResponse<WSMetrics>> {
    let metrics = state.ws_connection_manager.get_metrics().await;
    ResponseJson(ApiResponse::success(metrics))
}
```

### 2. 调试日志

```rust
// 使用 tracing 进行结构化日志
#[tracing::instrument(skip(app_state))]
async fn handle_subscribe(
    socket: SocketRef,
    data: SubscribeRequest,
    app_state: Arc<AppState>,
    client_id: String
) {
    tracing::info!(
        client_id = %client_id,
        symbol = %data.symbol,
        interval = %data.interval,
        "Processing kline subscription"
    );
    
    // 处理逻辑...
}
```

## 部署配置

### 1. 配置文件扩展

```toml
# config/default.toml 新增
[websocket]
enabled = true
max_connections = 10000
max_subscriptions_per_client = 50
heartbeat_interval = 30
message_batch_size = 100
message_batch_timeout_ms = 50

[websocket.cors]
enabled = true
allow_origins = ["*"]
```

### 2. 启动流程修改

```rust
// src/main.rs 修改
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // 现有初始化...
    
    // 创建 WebSocket 组件
    let ws_connection_manager = Arc::new(WSConnectionManager::new());
    let ws_broadcaster = Arc::new(WSBroadcaster::new(
        ws_connection_manager.clone(),
        config.websocket.message_batch_size,
        Duration::from_millis(config.websocket.message_batch_timeout_ms)
    ));
    let kline_manager = Arc::new(KlineManager::new(
        event_storage.clone(),
        ws_broadcaster.clone()
    ));
    
    // 启动后台任务
    tokio::spawn({
        let manager = ws_connection_manager.clone();
        async move {
            manager.start_health_checker().await;
        }
    });
    
    tokio::spawn({
        let broadcaster = ws_broadcaster.clone();
        async move {
            broadcaster.start_batch_processor().await;
        }
    });
    
    // 创建增强的应用状态
    let app_state = Arc::new(AppState {
        event_service,
        event_storage,
        ws_connection_manager,
        kline_manager,
        ws_broadcaster,
    });
    
    // 启动服务器
    let app = routes::create_router(&config, app_state);
    
    // 其余启动逻辑...
}
```

## 总结

该架构设计具有以下优势：

1. **高性能**: 使用 DashMap 和批处理优化并发性能
2. **可扩展**: 模块化设计，易于扩展新功能
3. **集成度高**: 充分利用现有 Solana 事件系统和 RocksDB 存储
4. **监控完善**: 内置性能指标和结构化日志
5. **配置灵活**: 支持环境特定配置
6. **前端友好**: 完全兼容 Socket.IO 客户端生态

这个方案既保持了与现有架构的兼容性，又提供了高性能的实时数据推送能力，非常适合生产环境使用。