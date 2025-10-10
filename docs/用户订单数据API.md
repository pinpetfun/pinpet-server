# 用户订单数据API文档

## 概述

本文档描述了新增的用户订单数据API，该API提供了按用户维度查询有效订单数据的功能。

## 功能特性

### 键值存储结构

- **键值前缀**: `uo:{user}:{mint}:{order_pda}`
- **存储内容**: 与现有订单数据完全一致
- **数据同步**: 与现有 `or:` 前缀的订单数据保持实时同步
- **查询优势**: 支持按用户+代币的精确查询，提高查询效率

### 支持的事件类型

1. **LongShort事件** → 创建用户订单数据
2. **PartialClose事件** → 更新用户订单数据  
3. **FullClose事件** → 删除用户订单数据
4. **ForceLiquidate事件** → 删除用户订单数据

## API端点

### 查询用户订单

**端点**: `GET /api/user_orders`

**查询参数**:
- `user` (必需): 用户地址
- `mint` (可选): 代币地址，用于精确查询特定代币的订单
- `page` (可选): 页码，默认为1
- `limit` (可选): 每页数量，默认为50，最大1000
- `order_by` (可选): 排序方式
  - `start_time_desc`: 按开始时间降序（默认）
  - `start_time_asc`: 按开始时间升序

**响应格式**:
```json
{
  "success": true,
  "data": {
    "orders": [
      {
        "order_type": 1,
        "mint": "token_address",
        "user": "user_address",
        "lock_lp_start_price": "1000000",
        "lock_lp_end_price": "1100000",
        "lock_lp_sol_amount": 1000000000,
        "lock_lp_token_amount": 1000000000,
        "start_time": 1640995200,
        "end_time": 1641081600,
        "margin_sol_amount": 500000000,
        "borrow_amount": 1000000000,
        "position_asset_amount": 1000000000,
        "borrow_fee": 100,
        "order_pda": "order_pda_address"
      }
    ],
            "total": 1,
        "user": "user_address",
        "mint_account": "token_address",
        "page": 1,
        "limit": 50,
        "has_next": false,
        "has_prev": false
  }
}
```

## 使用示例

### 基本查询
```bash
curl "http://localhost:3000/api/user_orders?user=user_address_123"
```

### 按代币精确查询
```bash
curl "http://localhost:3000/api/user_orders?user=user_address_123&mint=token_address_456"
```

### 分页查询
```bash
curl "http://localhost:3000/api/user_orders?user=user_address_123&page=2&limit=20"
```

### 按时间排序
```bash
# 按开始时间升序（最早到最晚）
curl "http://localhost:3000/api/user_orders?user=user_address_123&order_by=start_time_asc"

# 按开始时间降序（最晚到最早，默认）
curl "http://localhost:3000/api/user_orders?user=user_address_123&order_by=start_time_desc"
```

### 组合查询
```bash
# 查询特定用户在特定代币上的订单，按时间升序排列
curl "http://localhost:3000/api/user_orders?user=user_address_123&mint=token_address_456&order_by=start_time_asc&page=1&limit=100"
```

## 错误处理

### 参数验证错误
```json
{
  "success": false,
  "message": "user parameter cannot be empty"
}
```

```json
{
  "success": false,
  "message": "limit cannot exceed 1000"
}
```

```json
{
  "success": false,
  "message": "order_by must be 'start_time_asc' or 'start_time_desc'"
}
```

### 服务器错误
```json
{
  "success": false,
  "message": "Internal server error"
}
```

## 性能特点

1. **高效查询**: 使用用户维度的键前缀，避免跨mint查询
2. **精确查询**: 支持按用户+代币的精确查询，进一步提高查询效率
3. **实时同步**: 与订单事件处理保持同步，确保数据一致性
4. **分页支持**: 支持大数据集的高效分页查询
5. **排序优化**: 默认按 `start_time` 降序排列，符合业务需求

## 数据一致性保证

- 所有用户订单操作都在同一个事务中处理
- 使用RocksDB的WriteBatch确保原子性
- 与现有订单数据完全同步，避免数据不一致

## 测试

运行测试脚本：
```bash
node test_user_orders_api.js
```

## 注意事项

1. 用户订单数据仅在相关事件发生时创建/更新/删除
2. 查询结果按 `start_time` 字段排序，默认降序（最新的在前）
3. 分页从1开始计数
4. 最大每页数量限制为1000条记录

## 更新日志

- **v1.0.0**: 初始版本，支持基本的用户订单查询功能
