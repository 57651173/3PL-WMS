# 3PL-WMS仓储管理系统

## 🌐 语言版本 / Language Versions
- [中文 (Chinese)](README.md) - 当前页面
- [English](README_EN.md)
- [Deutsch (German)](README_DE.md)
- [日本語 (Japanese)](README_JA.md)

---

## 📋 项目概述

本项目是一个企业级仓储管理系统的核心模块，专注于订单处理、库存管理、拣货作业和发货流程。系统采用现代化的架构设计，支持高并发、高可用的仓储作业场景，并严格遵循FIFO（先进先出）原则进行库存管理。


## 🔄 完整业务流程

### 第一阶段：订单处理与拣货明细生成

```
1. 订单提交 (STATUS_PEND)
   ↓
2. 订单审核通过 (STATUS_CONF)
   ↓
3. 预拣货排序 + FIFO库存冻结 (STATUS_PICK)
   ↓
4. 生成拣货明细（picking_no = null, status = pending）
```

**关键操作：**
- 订单信息验证
- FIFO库存预检查
- 状态流转控制

### 第二阶段：波次拣货与执行

```
5. 拣货规则按波次绑定 → 补充拣货单号 → 拣货员拣货 → 打包 → 发货
6. 支持多种波次策略（按时间、库位、客户、优先级等）
7. 智能合并相同SKU，优化拣货路径
8. 实时跟踪拣货进度
```


**核心特性：**
- **FIFO 库存冻结** - 按入库时间先进先出原则冻结批次库存
- **智能路径优化** - 按库位编码排序优化拣货路径
- **多维度库存管理** - 批次、库位、仓库三级库存统计
- **波次拣货** - 根据波次规则分配拣货单号

### 第三阶段：发货处理

```
9. 自动发货处理 (STATUS_SHIP)
    ↓
10. FIFO减少冻结库存
    ↓
11. 物流交接
    ↓
12. 客户签收 (STATUS_FINISH)
```

## 🛠️ 技术特性

### FIFO 库存管理（核心特性）

#### 1. FIFO原则实现

```php
// 所有库存操作都按入库时间排序，确保先进先出
->orderBy('received_at ASC, created_at ASC')
```

#### 2. FIFO库存冻结

- **自动分配冻结** - `autoAllocateStockFreeze()` 按FIFO原则自动选择批次进行冻结
- **指定库位冻结** - `freezeLocationStockByFifo()` 在指定库位内按FIFO原则冻结
- **批次追溯** - 完整记录每个批次的冻结情况和入库时间

```php
// 示例：FIFO库存冻结
$inventoryService->freezeOrderStock(
    'ORD20241215001',
    'WH01',
    'CUST001',
    [
        [
            'product_sku' => 'SKU001',
            'sku_barcode' => 'BAR001',
            'qty_ordered' => 100
        ]
    ],
    "订单库存冻结"
);
```

#### 3. FIFO库存释放

- **自动释放** - `autoReleaseStockFreeze()` 按FIFO原则释放冻结库存
- **批次匹配** - 优先释放最早冻结的批次
- **库存平衡** - 自动更新批次、库位、仓库三级库存统计

#### 4. FIFO库存扣减

- **自动扣减** - `autoDeductStockFromLocation()` 按FIFO原则扣减可用库存
- **发货扣减** - 发货时优先扣减最早入库的批次
- **成本核算** - 支持FIFO成本核算方法

#### 5. FIFO优势

- **库存周转优化** - 确保先入库的商品先被使用，避免库存积压
- **保质期管理** - 对于有保质期的商品，FIFO可以减少过期风险
- **成本核算准确** - 按入库顺序使用库存，成本核算更准确
- **库存追溯** - 完整的批次追溯链，便于质量问题排查
- **合规要求** - 满足某些行业的FIFO合规要求

#### 6. 适用场景

- **食品饮料** - 严格的保质期管理
- **医药行业** - 药品批次管理和有效期控制
- **化工产品** - 化学品的稳定性和安全性管理
- **电子产品** - 避免元器件老化和技术过时
- **服装纺织** - 季节性商品的库存周转

### Redis 优化

- **分布式锁** - 防止并发操作冲突
- **缓存机制** - 提升查询性能
- **事件发布** - 实时状态通知
- **进度跟踪** - 实时拣货进度

#### 并发控制机制

系统采用Redis分布式锁来确保高并发场景下的数据一致性：

```php
// 批次库存锁 - 最细粒度
$lockKey = "batch_stock_lock:{$warehouseCode}:{$customerCode}:{$productSku}:{$skuBarcode}:{$lotNumber}";

// 库位库存锁 - 中等粒度
$lockKey = "location_stock_lock:{$warehouseCode}:{$locationCode}:{$customerCode}:{$productSku}:{$skuBarcode}";

// 仓库库存锁 - 较粗粒度
$lockKey = "warehouse_stock_lock:{$warehouseCode}:{$customerCode}:{$productSku}:{$skuBarcode}";
```

**锁的特性：**
- **锁超时时间**：30秒，防止死锁
- **最大等待时间**：10秒，避免长时间阻塞
- **原子操作**：确保库存更新的原子性
- **负库存检查**：防止库存超卖
- **自动重试**：临时性错误自动重试

**并发安全保障：**
- 多个订单同时冻结库存时，按顺序执行
- 入库和出库操作并发时，数据保持一致
- 批次库存更新时，防止数据竞争
- 库存统计实时准确，无数据丢失

### 智能路径优化

```php
// 按库位编码排序，优化拣货路径
usort($details, function($a, $b) {
    return strcmp($a['location_code'], $b['location_code']);
});
```

## 📊 使用示例

### 1. FIFO库存冻结示例

```php
$inventoryService = new InventoryService();

// 查看当前库存状态（按FIFO排序）
$stockStatus = $inventoryService->getDetailedStockStatus('WH01', 'CUST001', 'SKU001', 'BAR001');

// 执行FIFO库存冻结
$result = $inventoryService->freezeOrderStock(
    'ORD20241215001',
    'WH01',
    'CUST001',
    [
        [
            'product_sku' => 'SKU001',
            'sku_barcode' => 'BAR001',
            'qty_ordered' => 50
        ]
    ],
    "FIFO库存冻结演示"
);

// 系统会自动按入库时间先进先出的原则选择批次进行冻结
```

### 2. 订单处理阶段

```php
$orderService = new OrderService();
$result = $orderService->performPrePickSorting('SO202412150001', true);

// 返回结果包含：
// - 生成拣货明细（遵循FIFO原则）
```

### 3. 波次拣货阶段

```php
$waveService = new WavePickingService();
$waveResult = $waveService->generateWavePickingBatch([
    'strategy' => WavePickingService::STRATEGY_TIME,
    'warehouse_code' => 'WH01',
    'max_orders' => 10,
    'max_items' => 50
], [
    'picker' => 'picker001',
    'optimize_path' => true
]);

// 返回结果包含：
// - 生成拣货单
```

### 4. 手动分配明细

```php
$waveService = new WavePickingService();
$assignResult = $waveService->assignDetailsToPickingManually(
    [1, 2, 3], // 明细ID数组
    'WWH0120241215000001' // 拣货单号
);

// 返回结果包含：
// - 分配成功明细数量
```

### 5. 拣货执行阶段

```php
$pickingService = new PickingService();
$pickingService->startPicking('WWH0120241215000001', 'picker001');
$pickingService->scanAndPick('WWH0120241215000001', 'SKU001', 10, 'BATCH001');
$pickingService->completePicking('WWH0120241215000001');
```

## 🔧 配置选项

### 部分发货配置

```php
$config = [
    'enabled' => true,                    // 启用部分发货
    'min_completion_rate' => 0.8,         // 最小完成率 80%
    'max_partial_times' => 3,             // 最大部分发货次数
    'auto_ship_threshold' => 0.9,         // 自动发货阈值 90%
    'require_approval' => false,          // 无需审批
    'notification_enabled' => true,       // 启用通知
];

$orderService->setPartialShipmentConfig($config, 'CUSTOMER001', 'WH001');
```

### 拣货类型配置

```php
$pickingOptions = [
    'picking_type' => PickingService::TYPE_NORMAL,  // 普通拣货
    'picker' => 'picker001',                        // 指定拣货员
    'priority' => 1                                 // 优先级
];
```

## 📈 性能优化

### 1. Redis 缓存策略

- **订单信息缓存** - 5分钟 TTL
- **拣货进度缓存** - 30分钟 TTL
- **配置信息缓存** - 1小时 TTL
- **FIFO批次缓存** - 10分钟 TTL

### 2. 数据库优化

- **索引优化** - 关键字段建立复合索引
- **FIFO索引** - `(warehouse_code, customer_code, product_sku, received_at, created_at)`
- **批量操作** - 减少数据库往返次数
- **分页查询** - 大数据量分页处理

### 3. 并发控制

- **分布式锁** - 防止并发冲突
- **原子操作** - 确保数据一致性
- **重试机制** - 处理临时性错误

## 🚀 部署说明

### 环境要求

- PHP 7.4+
- Yii2 Framework
- MySQL 5.7+
- Redis 5.0+

## 🎯 未来规划

1. **AI 路径优化** - 机器学习优化拣货路径
2. **自动化集成** - 与自动化设备集成
3. **移动端支持** - 开发移动拣货应用
4. **数据分析** - 拣货效率分析报表
5. **多仓库支持** - 跨仓库调拨功能

## 📞 技术支持

如有问题或建议，请联系开发团队。

<img src="./docs/image/wechat.png" alt="donate" width="200" />

---

