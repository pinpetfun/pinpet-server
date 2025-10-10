# CPI 事件捕获实现说明

## 概述

本次修改实现了对 Solana 程序中通过 `emit!` 宏发出的所有事件的完整捕获，特别是解决了跨合约调用（CPI）中事件丢失的问题。

## 主要修改

### 1. 增强的事件监听器 (listener.rs)

#### 新增功能：
- **签名去重机制**：使用 `HashSet` 记录已处理的交易签名，避免重复处理
- **交易详情获取**：当检测到 CPI 调用时，主动获取完整交易详情
- **智能 CPI 检测**：通过日志中的 `invoke [2]`、`invoke [3]` 等标记识别 CPI 调用

#### 核心代码改动：
```rust
// 检测 CPI 调用
let has_cpi = logs.iter().any(|log| {
    log.contains("invoke [2]") || 
    log.contains("invoke [3]") ||
    log.contains("invoke [4]")
});

// 获取完整交易详情
if has_cpi {
    match client.get_transaction_with_logs(signature).await {
        Ok(tx_details) => {
            // 解析完整日志，包括 CPI 中的事件
        }
    }
}
```

### 2. 程序调用栈跟踪 (events.rs)

#### 新方法：`parse_events_with_call_stack`
- **调用栈追踪**：维护程序调用栈，准确识别事件来源
- **上下文感知**：只在目标程序上下文中解析 "Program data:" 日志
- **递归支持**：支持多层 CPI 调用的事件解析

#### 工作原理：
1. 监听 `Program <pubkey> invoke [depth]` 日志，将程序压入调用栈
2. 监听 `Program <pubkey> success/failed` 日志，将程序弹出调用栈
3. 只在目标程序处于调用栈中时解析 `Program data:` 日志

### 3. RPC 客户端增强 (client.rs)

#### 新方法：`get_transaction_with_logs`
- 获取交易的完整信息，包括所有日志
- 使用 `finalized` commitment 级别确保数据准确性
- 返回 JSON 格式便于解析

### 4. 事件去重机制

#### 实现方式：
- 基于签名和事件类型的去重
- 对于订单相关事件，还需要比较 `order_pda`
- 避免从不同来源（WebSocket 和 RPC）重复添加同一事件

## 技术亮点

### 1. 完整的 CPI 支持
- 不仅捕获顶层程序的事件
- 还能捕获嵌套在任意深度 CPI 调用中的事件
- 通过程序调用栈准确定位事件来源

### 2. 性能优化
- 只在检测到 CPI 时才获取完整交易
- 使用签名去重避免重复处理
- 批量处理事件减少开销

### 3. 可靠性保证
- 双通道获取（WebSocket + RPC）
- 事件去重确保数据一致性
- 详细的调试日志便于问题排查

## 使用说明

### 配置要求
确保 Solana 节点配置正确：
```toml
[solana]
rpc_url = "http://localhost:8899"
ws_url = "ws://localhost:8900"
program_id = "目标程序ID"
```

### 日志级别
建议开启 debug 级别查看详细的事件解析过程：
```toml
[logging]
level = "debug"
```

### 测试验证

1. **简单事件测试**：直接调用程序，验证事件捕获
2. **CPI 事件测试**：通过其他程序调用目标程序，验证 CPI 中的事件捕获
3. **多层 CPI 测试**：测试多层嵌套调用中的事件捕获

## 注意事项

1. **性能影响**：获取完整交易详情会增加 RPC 调用，可能影响性能
2. **网络延迟**：使用 `finalized` commitment 级别会有更高延迟
3. **存储开销**：去重机制需要在内存中维护已处理签名集合

## 后续优化建议

1. **定期清理**：定期清理已处理签名集合，避免内存无限增长
2. **批量获取**：批量获取多个交易详情，减少 RPC 调用次数
3. **缓存机制**：缓存交易详情，避免重复获取
4. **监控指标**：添加事件捕获率、CPI 检测率等监控指标

## 总结

本次实现完整解决了 Solana 程序事件捕获的核心问题，特别是 CPI 调用中的事件丢失问题。通过程序调用栈跟踪和智能 CPI 检测，确保所有通过 `emit!` 宏发出的事件都能被准确捕获和解析。

---

*实现日期：2025-09-11*
*版本：1.0.0*