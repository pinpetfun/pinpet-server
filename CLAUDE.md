# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 开发指南
  用中文聊天, 写注释用英文
  文档写到 notes 目录 用md格式, 有必要可以加图

### 常用命令

**构建和运行：**
- `cargo run` - 开发模式启动服务器
- `cargo run --release` - 生产模式启动服务器
- `cargo build` - 构建项目
- `cargo check` - 快速语法检查

**测试：**
- `./test.sh` - 运行完整的功能测试脚本，包括 API 接口测试
- `cargo test` - 运行单元测试

**开发工具：**
- `test_mint_details.js` - 测试代币详情 API
- `test_orders_api.js` - 测试订单 API
- `test_user_orders_api.js` - 测试用户订单 API
- `test_websocket.js` - 测试 WebSocket 连接

### 项目架构

**核心模块结构：**
- `src/main.rs` - 应用程序入口点，初始化配置、日志和服务
- `src/config.rs` - 配置管理，支持 TOML 配置文件
- `src/routes.rs` - API 路由定义
- `src/handlers/` - API 请求处理器
- `src/services/` - 业务逻辑服务层
- `src/solana/` - Solana 区块链交互模块
- `src/models.rs` - 数据模型定义

**Solana 集成架构：**
- `solana/client.rs` - Solana RPC 客户端封装
- `solana/listener.rs` - 区块链事件监听器
- `solana/events.rs` - 事件类型定义和解析

**数据存储层：**
- `services/event_storage.rs` - 事件数据存储接口
- RocksDB 作为本地键值存储，使用特定前缀组织数据：
  - `mt:` - Token markers
  - `tr:` - Transaction events
  - `or:` - Order data
  - `us:` - User events
  - `in:` - Token info

**配置系统：**
- `config/default.toml` - 默认配置 
- `config/production.toml` - 生产环境配置
- 支持环境变量覆盖配置项

### 开发约定

**代码风格：**
- 代码注释使用英文
- 日志输出使用英文
- 文档文件放在 `docs/` 目录，使用中文文件名
- 错误处理使用 `anyhow` 库

**API 设计：**
- 使用 Axum 框架构建 REST API
- 集成 Swagger UI 提供 API 文档（`/swagger-ui` 端点）
- 所有 API 响应使用统一的 JSON 格式
- 支持分页查询（page/limit 参数）

**事件处理流程：**
1. Solana 监听器接收区块链事件
2. 事件服务解析并分类处理不同类型的事件
3. RocksDB 存储事件数据，使用键值前缀分类
4. API 服务提供查询接口供前端访问

### 依赖关系

**主要依赖：**
- `axum` - Web 框架
- `tokio` - 异步运行时
- `serde/serde_json` - 序列化
- `config` - 配置管理
- `tracing` - 日志系统
- `rocksdb` - 数据存储
- `solana-client/solana-sdk` - Solana 区块链集成
- `utoipa` - OpenAPI/Swagger 文档生成

### 服务启动流程

1. 加载配置文件（支持环境特定配置）
2. 初始化日志系统
3. 创建事件服务（包含 RocksDB 和 Solana 监听器）
4. 启动 Axum Web 服务器
5. 如果启用，开始监听 Solana 事件

### 故障排查

**常见问题：**
- 如果事件监听器初始化失败，服务器会以禁用事件监听器的模式继续运行
- RocksDB 数据存储在 `./data/rocksdb` 目录
- 服务器默认监听 `0.0.0.0:8080`
- Solana RPC 连接配置在 `config/default.toml` 中

**调试工具：**
- 查看 `/api/events/status` 获取事件服务状态
- 查看 `/api/events/db-stats` 获取数据库统计信息
- 使用 `test.sh` 脚本验证所有 API 端点