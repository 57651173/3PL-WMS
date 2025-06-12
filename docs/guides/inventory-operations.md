# 海外仓库存管理系统 - 完整操作指南

## 概述

本文档详细介绍了海外仓库存管理系统中所有可用的库存操作，包括核心业务流程和扩展功能。系统采用严谨的事务管理和Redis分布式锁机制，确保高并发环境下的数据一致性。

## 📋 核心库存操作列表

### 🚀 基础业务流程 (6项核心操作)

| 序号 | 操作类型 | 功能说明 | 库存影响 | 状态变更 |
|------|----------|----------|----------|----------|
| 1 | **入库单提交** | 增加在途库存 | 在途库存 +N | 商品状态：在途 |
| 2 | **入库收货上架** | 在途转可用库存 | 在途库存 -N，可用库存 +N | 商品状态：可用 |
| 3 | **订单提交** | 冻结可用库存 | 可用库存 -N，冻结库存 +N | 订单状态：已确认 |
| 4 | **订单发货** | 减少冻结库存 | 冻结库存 -N | 订单状态：已发货 |
| 5 | **取消入库单** | 减少在途库存 | 在途库存 -N | 入库单：已取消 |
| 6 | **取消订单** | 释放冻结库存 | 冻结库存 -N，可用库存 +N | 订单状态：已取消 |

### 🔧 扩展管理功能 (12项高级操作)

| 序号 | 操作类型 | 功能说明 | 使用场景 | 库存影响 |
|------|----------|----------|----------|----------|
| 7 | **库存调整** | 盘盈盘亏处理 | 定期盘点 | 可用库存 ±N |
| 8 | **库存转移** | 仓库间/库位间转移 | 调拨优化 | 转出库存 -N，转入库存 +N |
| 9 | **库存损坏标记** | 标记损坏商品 | 质量问题 | 可用库存 -N，损坏库存 +N |
| 10 | **库存修复处理** | 损坏商品恢复 | 修复完成 | 损坏库存 -N，可用库存 +N |
| 11 | **库存预留** | 临时预留功能 | 特殊需求 | 可用库存 -N，预留库存 +N |
| 12 | **释放预留** | 预留库存释放 | 预留到期 | 预留库存 -N，可用库存 +N |
| 13 | **库存报废** | 无法使用商品处理 | 过期/损坏 | 损坏库存 -N |
| 14 | **过期库存查询** | 临期商品管理 | 先进先出 | 查询结果 |
| 15 | **批量更新** | 高效批量操作 | 大量调整 | 批量库存变更 |
| 16 | **库存汇总查询** | 总量统计 | 报表分析 | 查询结果 |
| 17 | **缓存查询** | 高性能查询 | 频繁查询 | 查询结果 |
| 18 | **详细状态查询** | 多维度库存状态 | 详细分析 | 查询结果 |

## 系统架构特点

### 1. 分布式锁机制
- 使用Redis实现分布式锁，防止并发操作冲突
- 自动超时释放，防止死锁
- 降级处理：Redis不可用时仍能正常工作

### 2. 缓存优化
- 库存数据缓存，提高查询性能
- 智能缓存失效机制
- 实时事件推送

### 3. 事务管理
- 数据库事务确保操作原子性
- 异常自动回滚
- 完整的审计日志

## 库存操作分类

### 基础业务流程 (6大核心功能)

#### 1. 入库单提交 - 增加在途库存
**场景**: 入库单从草稿状态提交为正式入库单
**影响**: 增加在途库存 (`stock_onway`)
**前提**: 无
**限制**: 无

```php
$inventoryService->addOnwayStock($inboundNo, $warehouseCode, $customerCode, $items, $remark);
```

#### 2. 入库收货上架 - 在途转可用
**场景**: 货物实际到达仓库并上架
**影响**: 减少在途库存，增加可用库存
**前提**: 必须有对应的在途库存
**限制**: 收货数量不能超过在途数量

```php
$inventoryService->receiveInboundStock($inboundNo, $warehouseCode, $customerCode, $items, $remark);
```

#### 3. 订单提交 - 冻结库存
**场景**: 订单从草稿状态提交为正式订单
**影响**: 减少可用库存，增加冻结库存
**前提**: 必须有足够的可用库存
**限制**: 不能超卖

```php
$inventoryService->freezeOrderStock($orderNo, $warehouseCode, $customerCode, $items, $remark);
```

#### 4. 订单发货 - 减少冻结库存
**场景**: 订单实际发货出库
**影响**: 减少冻结库存
**前提**: 必须有对应的冻结库存
**限制**: 发货数量不能超过冻结数量

```php
$inventoryService->shipOrderStock($orderNo, $warehouseCode, $customerCode, $items, $remark);
```

#### 5. 取消入库单 - 减少在途库存
**场景**: 取消未收货的入库单
**影响**: 减少在途库存
**前提**: 入库单尚未收货
**限制**: 只能取消未收货的入库单

```php
$inventoryService->cancelInboundStock($inboundNo, $warehouseCode, $customerCode, $items, $remark);
```

#### 6. 取消订单 - 释放冻结库存
**场景**: 取消未发货的订单
**影响**: 减少冻结库存，增加可用库存
**前提**: 订单尚未发货
**限制**: 只能取消未发货的订单

```php
$inventoryService->releaseOrderStock($orderNo, $warehouseCode, $customerCode, $items, $remark);
```

### 扩展管理功能 (完整的18项操作)

#### 7. 库存调整 - 盘盈盘亏处理
**场景**: 定期盘点发现的库存差异
**影响**: 直接调整可用库存数量
**类型**: 
- 盘盈：增加库存 (`qty_adjustment > 0`)
- 盘亏：减少库存 (`qty_adjustment < 0`)

```php
$items = [
    [
        'product_sku' => 'SKU001',
        'sku_barcode' => 'BARCODE001',
        'qty_adjustment' => 10,  // 正数盘盈，负数盘亏
        'reason' => '盘点发现多出10件'
    ]
];
$inventoryService->adjustStock($adjustmentNo, $warehouseCode, $customerCode, $items, $remark);
```

#### 8. 库存转移 - 仓库间/库位间转移
**场景**: 库存在不同仓库或库位之间转移
**影响**: 
- 源仓库：减少可用库存
- 目标仓库：增加可用库存
**限制**: 源仓库必须有足够库存

```php
$items = [
    [
        'product_sku' => 'SKU001',
        'sku_barcode' => 'BARCODE001',
        'qty_transfer' => 50,
        'from_location' => 'A01-001',
        'to_location' => 'B02-003'
    ]
];
$inventoryService->transferStock($transferNo, $fromWarehouse, $toWarehouse, $customerCode, $items, $remark);
```

#### 9. 库存损坏标记
**场景**: 商品在仓储过程中发生损坏
**影响**: 减少可用库存，增加损坏库存
**维度更新**: 仓库维度 + 库位维度 + 批次维度
**限制**: 不能超过可用库存数量

```php
$items = [
    [
        'product_sku' => 'SKU001',
        'sku_barcode' => 'BARCODE001',
        'qty_damaged' => 5,
        'location_code' => 'A01-001',      // 必须指定库位编码
        'batch_no' => 'BATCH20231201',     // 可选：批次号
        'damage_reason' => '包装破损',     // 损坏原因
        'damage_level' => 'light'          // 损坏等级：light/medium/severe
    ]
];
$inventoryService->markStockDamaged($damageNo, $warehouseCode, $customerCode, $items, $remark);
```

#### 10. 库存修复处理
**场景**: 损坏商品经过处理后恢复可用
**影响**: 减少损坏库存，增加可用库存
**维度更新**: 仓库维度 + 库位维度 + 批次维度
**限制**: 不能超过损坏库存数量

```php
$items = [
    [
        'product_sku' => 'SKU001',
        'sku_barcode' => 'BARCODE001',
        'qty_repaired' => 3,
        'location_code' => 'A01-001',      // 必须指定库位编码
        'batch_no' => 'BATCH20231201',     // 可选：批次号
        'repair_method' => '重新包装',     // 修复方法
        'quality_check' => 'passed'        // 质检结果：passed/failed
    ]
];
$inventoryService->repairDamagedStock($repairNo, $warehouseCode, $customerCode, $items, $remark);
```

#### 11. 库存预留功能
**场景**: 
- 促销活动预留库存
- VIP客户专属库存
- 特殊订单临时预留

```php
// 预留库存
$items = [
    [
        'product_sku' => 'SKU001',
        'sku_barcode' => 'BARCODE001',
        'qty_reserved' => 100,
        'location_code' => 'A01-001',
        'expire_time' => '2023-12-31 23:59:59'
    ]
];
$inventoryService->reserveStock($reserveNo, $warehouseCode, $customerCode, $items, $remark);

// 释放预留
$inventoryService->releaseReservedStock($reserveNo, $warehouseCode, $customerCode, $items, $remark);
```

#### 12. 库存报废处理
**场景**: 
- 无法修复的损坏商品
- 过期产品处理
- 质量不合格产品销毁

```php
$items = [
    [
        'product_sku' => 'SKU001',
        'sku_barcode' => 'BARCODE001',
        'qty_scrapped' => 5,
        'location_code' => 'A01-001',
        'scrap_reason' => '过期无法使用'
    ]
];
$inventoryService->scrapStock($scrapNo, $warehouseCode, $customerCode, $items, $remark);
```

#### 13. 批次管理功能
**应用场景**:
- 生产日期追踪
- 有效期管理
- 批次合并/拆分

```php
// 获取即将过期的库存
$expiringStock = $inventoryService->getExpiringStock($warehouseCode, $customerCode, $days = 30);

// 批次合并（需要额外实现）
$inventoryService->mergeBatches($mergeNo, $warehouseCode, $customerCode, $sourceBatches, $targetBatch);

// 批次拆分（需要额外实现）
$inventoryService->splitBatch($splitNo, $warehouseCode, $customerCode, $sourceBatch, $targetBatches);
```

#### 14. 退货处理功能
**应用场景**:
- 客户退货入库
- 供应商退货出库
- 质量问题退货

```php
// 退货入库（可使用现有入库流程）
$inventoryService->receiveInboundStock($returnInboundNo, $warehouseCode, $customerCode, $items, '退货入库');

// 退货出库（可使用现有出库流程）
$inventoryService->shipOrderStock($returnOutboundNo, $warehouseCode, $customerCode, $items, '退货出库');
```

#### 15. 库存冻结/解冻功能
**应用场景**:
- 质量检查期间冻结
- 法律纠纷期间冻结
- 安全检查冻结

```php
// 冻结库存（使用现有冻结功能）
$inventoryService->freezeOrderStock($freezeNo, $warehouseCode, $customerCode, $items, '质量检查冻结');

// 解冻库存（使用释放功能）
$inventoryService->releaseOrderStock($freezeNo, $warehouseCode, $customerCode, $items, '质量检查通过');
```

#### 16. 库存盘点功能
**应用场景**:
- 定期盘点
- 循环盘点
- 随机抽盘

```php
// 盘点调整（使用现有调整功能）
$inventoryService->adjustStock($countNo, $warehouseCode, $customerCode, $items, '盘点调整');
```

#### 17. 库存分配功能
**应用场景**:
- 多订单库存分配
- 优先级分配
- 智能分配算法

```php
// 可通过预留功能实现分配
$inventoryService->reserveStock($allocationNo, $warehouseCode, $customerCode, $items, '库存分配');
```

#### 18. 批量库存更新
**应用场景**:
- 定时任务批量更新
- 系统集成批量同步
- 数据修复批量处理

```php
$updates = [
    [
        'product_sku' => 'SKU001',
        'sku_barcode' => 'BARCODE001',
        'changes' => ['stock_available' => 10, 'stock_damaged' => -5]
    ],
    [
        'product_sku' => 'SKU002',
        'sku_barcode' => 'BARCODE002',
        'changes' => ['stock_available' => -20, 'stock_freeze' => 20]
    ]
];
$inventoryService->batchUpdateStock($warehouseCode, $customerCode, $updates);
```

## 查询和监控功能

### 实时库存查询
```php
// 单个商品库存状态（带缓存）
$stock = $inventoryService->getStockStatusWithCache($warehouseCode, $customerCode, $productSku, $skuBarcode);

// 获取库存总计
$totalStock = $inventoryService->getTotalStock($warehouseCode, $customerCode, $productSku, $skuBarcode);

// 批量查询库存状态
$stocks = $inventoryService->batchGetStockStatus($warehouseCode, $customerCode, $products);

// 详细库存状态（包含库位和批次分布）
$detailedStock = $inventoryService->getDetailedStockStatus($warehouseCode, $customerCode, $productSku, $skuBarcode);
```

### 库位维度查询
```php
// 获取库位可用库存
$locationAvailable = $inventoryService->getLocationAvailableStock($warehouseCode, $locationCode, $customerCode, $productSku, $skuBarcode);

// 获取库位损坏库存
$locationDamaged = $inventoryService->getLocationDamagedStock($warehouseCode, $locationCode, $customerCode, $productSku, $skuBarcode);

// 获取库位冻结库存
$locationFreeze = $inventoryService->getLocationFreezeStock($warehouseCode, $locationCode, $customerCode, $productSku, $skuBarcode);
```

### 库存预警和监控
```php
// 低库存预警
$lowStockProducts = $inventoryService->getLowStockAlert($warehouseCode, $customerCode, $threshold);

// 零库存商品
$zeroStockProducts = $inventoryService->getZeroStockProducts($warehouseCode, $customerCode);

// 损坏库存查询
$damagedStock = $inventoryService->getDamagedStock($warehouseCode, $customerCode, $productSku, $skuBarcode);

// 过期库存预警
$expiringStocks = $inventoryService->getExpiringStock($warehouseCode, $customerCode, $days);
```

### 数据一致性校验
```php
// 单个商品一致性校验
$validation = $inventoryService->validateStockConsistency($warehouseCode, $customerCode, $productSku, $skuBarcode);

// 批量一致性校验（需要额外实现）
$batchValidation = $inventoryService->batchValidateStockConsistency($warehouseCode, $customerCode, $products);
```

### 统计和报表
```php
// 库存汇总报表
$summary = $inventoryService->getStockSummaryReport($warehouseCode, $customerCode);

// 库存变动历史
$history = $inventoryService->getStockTransactionHistory($warehouseCode, $customerCode, $productSku, $transactionType, $limit);

// 库存周转分析（需要额外实现）
$turnover = $inventoryService->getStockTurnoverAnalysis($warehouseCode, $customerCode, $dateRange);
```

## 完整功能列表

### 已实现的核心功能 (22项)
1. **addOnwayStock** - 入库单提交增加在途库存
2. **receiveInboundStock** - 入库收货上架处理  
3. **freezeOrderStock** - 订单提交冻结库存
4. **shipOrderStock** - 订单发货减少冻结库存
5. **cancelInboundStock** - 取消入库单减少在途库存
6. **releaseOrderStock** - 取消订单释放冻结库存
7. **adjustStock** - 库存调整（盘盈盘亏）
8. **transferStock** - 库存转移（仓库间/库位间）
9. **markStockDamaged** - 库存损坏标记
10. **repairDamagedStock** - 库存修复处理
11. **reserveStock** - 库存预留功能
12. **releaseReservedStock** - 释放预留库存
13. **scrapStock** - 库存报废处理
14. **getExpiringStock** - 获取即将过期库存
15. **batchUpdateStock** - 批量更新库存状态
16. **getTotalStock** - 获取库存总计
17. **getStockStatusWithCache** - 缓存库存查询
18. **getDetailedStockStatus** - 详细库存状态
19. **validateStockConsistency** - 数据一致性校验
20. **getStockSummaryReport** - 库存汇总报表
21. **getStockTransactionHistory** - 库存变动历史
22. **publishStockChangeEvent** - 发布库存变更事件

### 查询功能 (10项)
1. **getAvailableStock** - 获取可用库存
2. **getFreezeStock** - 获取冻结库存
3. **getOnwayStock** - 获取在途库存
4. **getDamagedStock** - 获取损坏库存
5. **getLocationAvailableStock** - 获取库位可用库存
6. **getLocationDamagedStock** - 获取库位损坏库存
7. **getLocationFreezeStock** - 获取库位冻结库存
8. **batchGetStockStatus** - 批量查询库存状态
9. **getLowStockAlert** - 低库存预警
10. **getZeroStockProducts** - 零库存商品

## Redis集成功能

### 分布式锁
- 防止并发操作冲突
- 自动超时机制
- 降级处理支持

### 缓存机制
- 库存数据缓存
- 智能缓存失效
- 多级缓存策略

### 实时事件推送
```php
// 订阅库存变更事件
$redis->psubscribe(['stock_change_events'], function($redis, $pattern, $channel, $message) {
    $event = json_decode($message, true);
    // 处理库存变更事件
    handleStockChangeEvent($event);
});
```

## 最佳实践

### 1. 并发控制
- 所有库存变更操作都使用分布式锁
- 合理设置锁超时时间
- 实现锁等待和重试机制

### 2. 数据一致性
- 定期执行一致性校验
- 异常情况下的自动修复
- 完整的审计日志记录

### 3. 性能优化
- 合理使用缓存机制
- 批量操作优化
- 异步处理非关键操作

### 4. 监控和预警
- 实时库存监控
- 异常情况预警
- 性能指标跟踪

### 5. 错误处理
- 完善的异常处理机制
- 详细的错误日志记录
- 优雅的降级处理

## 配置要求

### Redis配置
```php
// common/config/main.php
'components' => [
    'redis' => [
        'class' => 'yii\redis\Connection',
        'hostname' => 'localhost',
        'port' => 6379,
        'database' => 0,
    ],
    'cache' => [
        'class' => 'yii\redis\Cache',
        'redis' => 'redis'
    ]
]
```

### 数据库优化
- 合适的索引策略
- 分区表支持大数据量
- 读写分离配置

## 安全考虑

### 1. 权限控制
- 操作权限验证
- 敏感操作审计
- 数据访问控制

### 2. 数据保护
- 重要操作备份
- 误操作恢复机制
- 数据加密存储

### 3. 系统监控
- 异常操作监控
- 性能指标跟踪
- 安全事件记录

## 扩展开发指南

### 1. 新增库存操作类型
```php
// 1. 添加新的事务类型常量
const TRANSACTION_TYPE_NEW_OPERATION = 'new_operation';

// 2. 实现具体操作方法
public function newStockOperation($params) {
    $lockKey = "new_operation:{$warehouseCode}:{$customerCode}";
    return $this->withDistributedLock($lockKey, function() use ($params) {
        // 实现具体逻辑
    });
}

// 3. 更新事务编号生成逻辑
// 4. 添加相应的模型和数据表支持
```

### 2. 集成第三方系统
- ERP系统对接
- WMS系统集成
- 电商平台同步

### 3. 自定义业务规则
- 库存预警规则
- 自动补货策略
- 库存分配算法

这个增强版的库存管理系统提供了完整的库存生命周期管理，具备高并发处理能力、严格的数据一致性保证，以及丰富的监控和分析功能，能够满足复杂海外仓业务场景的需求。 