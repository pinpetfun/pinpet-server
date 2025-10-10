# RocksDB 键值结构说明

## 键值格式概览

本项目使用 RocksDB 作为高性能的键值存储引擎，所有数据都按照特定的键格式进行组织。以下是所有使用的键格式：

| 键前缀 | 完整格式 | 描述 |
|-------|---------|------|
| `mt:` | `mt:{slot:010}:{mint_account}` | 代币标记，按slot排序索引 |
| `tr:` | `tr:{mint_account}:{slot:010}:{event_type}:{signature}` | 交易事件数据 |
| `or:` | `or:{mint_account}:up:{order_pda}` | 做多订单数据 |
| `or:` | `or:{mint_account}:dn:{order_pda}` | 做空订单数据 |
| `us:` | `us:{user}:{mint_account}:{slot}` | 用户交易事件 |
| `uo:` | `uo:{user}:{mint_account}:{order_pda}` | 用户订单数据 |
| `in:` | `in:{mint_account}` | 代币基本信息 |
| `s1:` | `s1:{mint_account}:{010}` | 1秒级的K线数据 |
| `s30:` | `m1:{mint_account}:{010}` | 30秒级的K线数据 |
| `m5:` | `m5:{mint_account}:{010}` | 5分钟级的K线数据 |



## 前缀说明

- **tr**: trade（交易）- 存储所有交易事件数据
- **or**: order（订单）- 存储订单相关数据
- **us**: user（用户）- 存储用户相关数据
- **uo**: user order（用户订单）- 存储用户订单数据
- **mt**: mint（铸币）- 按slot排序的代币索引
- **in**: info（信息）- 存储代币基本信息

## 详细说明

### 1. 代币标记键 (mt:)

格式: `mt:{slot:010}:{mint_account}`

- **用途**: 按创建时间（slot）排序的代币索引，支持高效的时间顺序查询和分页
- **值**: 空值，键本身包含所有必要信息
- **slot格式**: 10位数字，前补0，确保字典序等于数值序
- **排序支持**: 
  - 升序：直接迭代 `mt:` 前缀（最早创建的在前）
  - 降序：反向迭代 `mt:` 前缀（最新创建的在前）
- **访问方式**: 通过 `/api/mints?sort_by=slot_asc` 或 `sort_by=slot_desc` 查询
- **高效分页**: 利用RocksDB天然的有序性，无需加载全部数据即可分页
- **示例**: `mt:0123456789:CiP5VTLBWLYDfXXDYeMAbsRM4W2qw1NZ3XbCmn5qJjhf`

### 2. 交易事件键 (tr:)

格式: `tr:{mint_account}:{slot}:{event_type}:{signature}`

- **用途**: 存储所有交易事件数据
- **值**: 序列化后的事件数据（JSON）
- **事件类型**:
  - `tc`: 代币创建事件
  - `bs`: 买卖事件
  - `ls`: 做多/做空事件
  - `fl`: 强制清算事件
  - `fc`: 完全关闭事件
  - `pc`: 部分关闭事件
- **访问方式**: 通过 `/api/events` API 端点查询
- **示例**: `tr:CiP5VTLBWLYDfXXDYeMAbsRM4W2qw1NZ3XbCmn5qJjhf:123456789:bs:5xjP8qT7KKbGy9SHVf`

### 3. 订单键 (or:)

格式: 
- 做多订单: `or:{mint_account}:up:{order_pda}`
- 做空订单: `or:{mint_account}:dn:{order_pda}`

- **用途**: 存储做多/做空订单数据
- **值**: 序列化后的订单数据（JSON）
- **访问方式**: 通过 `/api/mint_orders` API 端点查询
- **示例**:
  - 做多订单: `or:CiP5VTLBWLYDfXXDYeMAbsRM4W2qw1NZ3XbCmn5qJjhf:up:Hx7J9sJ2U5CX6m7sMpF`
  - 做空订单: `or:CiP5VTLBWLYDfXXDYeMAbsRM4W2qw1NZ3XbCmn5qJjhf:dn:K9mP2sC7XvR8dLpQqT`

### 4. 用户交易键 (us:)

格式: `us:{user}:{mint_account}:{slot}`

- **用途**: 存储用户的所有交易事件
- **值**: 序列化后的用户交易数据（JSON）
- **访问方式**: 通过 `/api/user_event` API 端点查询
- **示例**: `us:6FpBZJcY3mJiV8U8zdQvvBs5QC9EJ2JeieRr1TW4QD6V:CiP5VTLBWLYDfXXDYeMAbsRM4W2qw1NZ3XbCmn5qJjhf:123456789`

### 5. 用户订单键 (uo:)

格式: `uo:{user}:{mint_account}:{order_pda}`

- **用途**: 存储用户的有效订单数据，支持按用户维度查询订单
- **值**: 序列化后的订单数据（JSON），与 `or:` 键的数据格式完全一致
- **数据同步**: 与 `or:` 前缀的订单数据保持实时同步
- **查询优势**:
  - 支持按用户查询所有订单
  - 支持按用户+代币精确查询
  - 避免跨mint查询的性能问题
- **访问方式**: 通过 `/api/user_orders` API 端点查询
- **查询参数**:
  - `user` (必需): 用户地址
  - `mint` (可选): 代币地址，用于精确查询
- **示例**: `uo:6FpBZJcY3mJiV8U8zdQvvBs5QC9EJ2JeieRr1TW4QD6V:CiP5VTLBWLYDfXXDYeMAbsRM4W2qw1NZ3XbCmn5qJjhf:Hx7J9sJ2U5CX6m7sMpF`

### 6. 代币信息键 (in:)

格式: `in:{mint_account}`

- **用途**: 存储代币的基本信息，用于首页列表、推荐列表等
- **值**: 序列化后的代币信息数据（JSON）
- **包含信息**:
  - 代币名称
  - 符号
  - URI
  - 创建时间
  - 最新价格
  - 交易统计数据
- **访问方式**: 通过 `/api/details` API 端点查询
- **示例**: `in:CiP5VTLBWLYDfXXDYeMAbsRM4W2qw1NZ3XbCmn5qJjhf`

## 查询示例

以下是几个常见查询场景的键格式示例：

1. **查询特定代币的所有事件**:
   - 前缀: `tr:{mint_account}:`
   
2. **查询特定用户的所有交易**:
   - 前缀: `us:{user}:`
   
3. **查询特定代币的做多订单**:
   - 前缀: `or:{mint_account}:up:`
   
4. **查询特定代币的做空订单**:
   - 前缀: `or:{mint_account}:dn:`

5. **查询特定用户的所有订单**:
   - 前缀: `uo:{user}:`
   
6. **查询特定用户在特定代币上的订单**:
   - 前缀: `uo:{user}:{mint_account}:`


