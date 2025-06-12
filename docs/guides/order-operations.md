# 订单处理操作指南

完整的订单处理服务说明文档，涵盖订单生命周期管理、Redis优化特性和最佳实践。

## 📋 目录

1. [订单处理核心功能](#订单处理核心功能)
2. [Redis优化特性](#redis优化特性)
3. [部分出单配置管理](#部分出单配置管理)
4. [部分出单支持](#部分出单支持)
5. [订单状态流转](#订单状态流转)
6. [订单处理特性](#订单处理特性)
7. [快速使用示例](#快速使用示例)
8. [配置管理](#配置管理)
9. [性能监控](#性能监控)
10. [最佳实践](#最佳实践)

---

## 订单处理核心功能

### 🎯 主要功能模块

1. **预拣货排序** - 智能拣货路径优化
   - 根据库位分布优化拣货顺序
   - 智能库位分配算法
   - 缺货商品预警和处理

2. **发货减库存** - 出库操作自动化（支持部分出单）
   - 完整发货和部分发货支持
   - 防超发机制和数量验证
   - 批次追溯和库位管理

3. **取消释放库存** - 订单取消库存恢复
   - 自动释放冻结库存
   - 状态回滚和一致性保证

4. **退货增加库存** - 退货商品重新入库
   - 支持完整退货和部分退货
   - 退货商品质量检查流程

5. **退货减少库存** - 损坏退货商品处理
   - 损坏商品单独处理
   - 退货原因分类统计

---

## Redis优化特性

### 🚀 核心优化功能

#### 1. 分布式锁机制
- **防并发冲突**：确保多服务器环境下的数据一致性
- **智能重试**：5次重试机制，100ms间隔
- **自动释放**：30秒TTL防止死锁
- **原子操作**：Lua脚本确保释放锁的原子性

```php
// 分布式锁示例
$lockKey = 'ship_order:' . $order_no;
$lockValue = $this->acquireLock($lockKey);
if (!$lockValue) {
    throw new Exception("订单正在处理中，请稍后重试");
}
```

#### 2. 订单信息缓存
- **性能提升**：查询性能提升3-10倍
- **智能缓存**：热点数据优先缓存
- **主动失效**：订单变更时自动清除缓存
- **降级机制**：Redis不可用时自动降级到数据库

#### 3. 动态配置管理
- **实时更新**：配置变更立即生效，无需重启服务
- **分层配置**：全局 → 订单类型 → 仓库 → 客户优先级
- **配置缓存**：1小时TTL的配置缓存机制
- **版本控制**：配置变更历史追踪

#### 4. 实时事件通知
- **Redis Pub/Sub**：实时事件推送机制
- **多系统同步**：支持与多个外部系统实时同步
- **异步处理**：避免阻塞主业务流程
- **消息持久化**：确保重要事件不丢失

---

## 部分出单配置管理

### 📋 配置系统架构

#### 分层配置优先级
```
客户配置 (优先级最高)
    ↓
仓库配置
    ↓  
订单类型配置
    ↓
全局配置 (优先级最低)
```

#### 核心配置选项

| 配置项 | 说明 | 默认值 | 示例 |
|--------|------|--------|------|
| `enabled` | 启用部分出单开关 | `true` | `true/false` |
| `min_completion_rate` | 最小完成率要求 | `0.3` | `0.0-1.0` |
| `max_partial_times` | 最大部分发货次数 | `5` | `1-10` |
| `auto_ship_threshold` | 自动发货阈值 | `0.8` | `0.0-1.0` |
| `require_approval` | 是否需要审批 | `false` | `true/false` |
| `notification_enabled` | 启用通知功能 | `true` | `true/false` |
| `priority_products` | 优先发货商品列表 | `[]` | `['SKU001', 'SKU002']` |
| `blacklist_products` | 禁止部分发货商品 | `[]` | `['FRAGILE001']` |

#### 配置示例

```php
// VIP客户专属配置
$vipConfig = [
    'enabled' => true,
    'min_completion_rate' => 0.1,        // VIP客户最小完成率降至10%
    'max_partial_times' => 10,           // 允许更多次部分发货
    'require_approval' => false,
    'notification_enabled' => true,
    'priority_products' => ['VIP001', 'VIP002', 'VIP003']
];

// 海外仓专属配置
$overseasConfig = [
    'enabled' => true,
    'min_completion_rate' => 0.5,        // 海外仓要求更高完成率
    'max_partial_times' => 3,            // 限制部分发货次数
    'require_approval' => true,          // 需要审批
    'blacklist_products' => ['DANGEROUS001', 'LIQUID001']
];
```

---

## 部分出单支持

### 📦 功能特性

#### 1. 灵活发货控制
- **精确数量控制**：支持指定每个商品的具体发货数量
- **批次指定发货**：可以指定具体批次和库位的分批发货
- **智能库存检查**：实时验证库存可用性
- **防超发机制**：严格控制累计发货数量不超过订单原始数量

#### 2. 智能状态管理
- **状态自动判断**：系统自动区分部分发货和完全发货状态
- **状态码定义**：
  - `75` - 部分发货状态
  - `80` - 完全发货状态
- **状态流转控制**：确保状态变更的合法性和一致性

#### 3. 批次追溯管理
- **完整追溯链**：记录每批货物的发货批次和库位信息
- **批次优先级**：支持FIFO/LIFO等批次选择策略
- **过期批次处理**：优先发货临期商品

#### 4. 完成度追踪
- **实时进度**：显示发货进度百分比
- **剩余商品统计**：详细显示未发货商品清单
- **完成预测**：基于历史数据预测完成时间

### 📊 发货控制示例

```php
// 部分发货示例
$shipItems = [
    [
        'product_sku' => 'PRODUCT001',
        'sku_barcode' => 'BAR001',
        'qty_shipped' => 50,              // 发货50个（总订单100个）
        'batch_no' => 'BATCH20231201',
        'location_code' => 'A001'
    ],
    [
        'product_sku' => 'PRODUCT002', 
        'sku_barcode' => 'BAR002',
        'qty_shipped' => 30,              // 发货30个（总订单50个）
        'batch_no' => 'BATCH20231202',
        'location_code' => 'B002'
    ]
];

$result = $orderService->shipOrderStock($orderNo, $shipItems, '第一次部分发货');
```

---

## 订单状态流转

### 🔄 完整状态流转图

```
订单创建 → 待审核(PEND)
    ↓
订单提交 → 已提交(CONF) [冻结库存]
    ↓
预拣货排序 → 已拣货(PICK)
    ↓
订单发货 → 部分发货(PARTIAL_SHIP) [首次部分发货]
    ↓            ↓
继续发货 → 已发货(SHIP) [完全发货完成]
    ↓
客户签收 → 已完成(FINISH)
```

### 📋 状态定义

| 状态码 | 状态名称 | 说明 | 允许操作 |
|--------|----------|------|----------|
| `10` | 待审核(PEND) | 订单创建待审核 | 审核、取消 |
| `20` | 已提交(CONF) | 订单审核通过 | 拣货、取消 |
| `30` | 已拣货(PICK) | 完成预拣货排序 | 发货、取消 |
| `75` | 部分发货(PARTIAL_SHIP) | 已部分发货 | 继续发货、取消 |
| `80` | 已发货(SHIP) | 完全发货完成 | 签收、退货 |
| `90` | 已完成(FINISH) | 客户已签收 | 退货 |
| `99` | 已取消(CANCEL) | 订单已取消 | 无 |

### 🔄 状态转换规则

#### 正向流转
- `PEND` → `CONF`：订单审核通过
- `CONF` → `PICK`：完成预拣货排序
- `PICK` → `PARTIAL_SHIP`：首次部分发货
- `PARTIAL_SHIP` → `SHIP`：完成全部发货
- `SHIP` → `FINISH`：客户签收确认

#### 逆向流转
- `CONF/PICK/PARTIAL_SHIP` → `CANCEL`：订单取消
- `SHIP/FINISH` → 退货处理（不改变订单状态，创建退货单）

---

## 订单处理特性

### 🔧 高级处理功能

#### 1. 智能库位分配
- **距离优化**：基于库位距离优化拣货路径
- **库存密度**：优先选择库存较多的库位
- **操作效率**：减少拣货员移动距离
- **批次管理**：结合批次规则进行库位选择

```php
// 库位分配示例
$sortedItems = $this->sortItemsByLocation($order, $orderItems);
foreach ($sortedItems as $item) {
    foreach ($item['locations'] as $location) {
        echo "商品：{$item['product_sku']} 库位：{$location['location_code']} 数量：{$location['pick_quantity']}\n";
    }
}
```

#### 2. 库存实时检查
- **可用性验证**：实时检查库存可用数量
- **预留冲突检测**：避免库存超卖
- **批次有效性**：检查批次是否过期
- **库位可达性**：确认库位可正常拣货

#### 3. 批次追溯管理
- **FIFO策略**：先进先出批次选择
- **临期优先**：优先出库临期商品
- **批次锁定**：防止批次混乱
- **追溯完整性**：保证可追溯到具体批次

#### 4. 退货灵活处理
- **部分退货**：支持订单商品的部分退货
- **质量分级**：区分正常退货和损坏退货
- **重新入库**：正常商品可重新入库销售
- **损坏处理**：损坏商品单独处理流程

---

## 快速使用示例

### 📝 基础操作示例

#### 1. 预拣货排序
```php
use common\services\OrderService;

$orderService = new OrderService();

// 执行预拣货排序
$result = $orderService->performPrePickSorting('XS-ORDER-20231201-001');

if ($result) {
    echo "预拣货排序完成：\n";
    echo "- 总商品种类：{$result['total_items']}\n";
    echo "- 涉及库位数：" . count($result['picking_locations']) . "\n";
    echo "- 库存充足商品：{$result['stock_check']['sufficient_stock']}\n";
    echo "- 库存不足商品：{$result['stock_check']['insufficient_stock']}\n";
}
```

#### 2. 完整发货
```php
// 完整发货（使用订单原始明细）
$result = $orderService->shipOrderStock(
    'XS-ORDER-20231201-001',
    [], // 空数组表示发货全部商品
    '正常发货'
);

if ($result['success']) {
    echo "发货成功：\n";
    echo "- 发货类型：{$result['shipment_type']}\n";
    echo "- 发货商品数：{$result['shipped_items']}\n";
    echo "- 完成率：{$result['shipment_summary']['completion_rate']}%\n";
}
```

#### 3. 部分发货
```php
// 部分发货（指定具体发货数量）
$shipItems = [
    [
        'product_sku' => 'PRODUCT001',
        'sku_barcode' => 'BAR001',
        'qty_shipped' => 50,              // 仅发货50个
        'batch_no' => 'BATCH20231201',
        'location_code' => 'A001'
    ]
];

$result = $orderService->shipOrderStock(
    'XS-ORDER-20231201-002',
    $shipItems,
    '第一次部分发货'
);

if ($result['success'] && $result['shipment_type'] === 'partial') {
    echo "部分发货成功：\n";
    echo "- 剩余未发货商品：{$result['remaining_items']}种\n";
    echo "- 当前完成率：{$result['shipment_summary']['completion_rate']}%\n";
}
```

#### 4. 订单取消
```php
// 取消订单并释放库存
$result = $orderService->cancelOrderStock(
    'XS-ORDER-20231201-003',
    '客户主动取消'
);

if ($result) {
    echo "订单取消成功，库存已释放\n";
} else {
    echo "订单取消失败\n";
}
```

#### 5. 退货处理
```php
// 正常退货（商品重新入库）
$returnItems = [
    [
        'product_sku' => 'PRODUCT001',
        'sku_barcode' => 'BAR001',
        'qty_returned' => 10,
        'location_code' => 'RETURN01',
        'reason' => '客户不满意'
    ]
];

$result = $orderService->returnOrderAddStock(
    'XS-ORDER-20231201-004',
    $returnItems,
    '正常退货重新入库'
);

// 损坏退货（商品不可再售）
$damagedItems = [
    [
        'product_sku' => 'PRODUCT002',
        'sku_barcode' => 'BAR002', 
        'qty_returned' => 5,
        'reason' => '运输途中损坏'
    ]
];

$result = $orderService->returnOrderReduceStock(
    'XS-ORDER-20231201-004',
    $damagedItems,
    '损坏商品减库存'
);
```

#### 6. 获取订单详情
```php
// 获取订单完整信息
$orderDetails = $orderService->getOrderFullDetails('XS-ORDER-20231201-001');

if ($orderDetails) {
    echo "订单信息：\n";
    echo "- 订单号：{$orderDetails['order']['order_no']}\n";
    echo "- 订单状态：{$orderDetails['order']['order_status']}\n";
    echo "- 商品种类：{$orderDetails['stock_status']['total_products']}\n";
    echo "- 库存充足：{$orderDetails['stock_status']['sufficient_stock']}\n";
    echo "- 库存不足：{$orderDetails['stock_status']['insufficient_stock']}\n";
    
    // 显示库存详情
    foreach ($orderDetails['stock_status']['stock_details'] as $detail) {
        $status = $detail['is_sufficient'] ? '✓' : '✗';
        echo "  {$status} {$detail['product_sku']}: 需要{$detail['qty_ordered']}个，可用{$detail['qty_available']}个\n";
    }
}
```

---

## 配置管理

### ⚙️ 动态配置系统

#### 1. 命令行配置工具

```bash
# 设置全局配置
./yii order-config/set-global true 0.3 5

# 设置VIP客户配置
./yii order-config/set-customer VIP_CUSTOMER_001 true 0.1 10

# 设置海外仓配置
./yii order-config/set-warehouse OVERSEAS_WH_001 true 0.5 true

# 设置紧急订单配置
./yii order-config/set-type URGENT false 1.0 true

# 查看配置
./yii order-config/get VIP_CUSTOMER_001 NORMAL_WH NORMAL

# 列出所有配置
./yii order-config/list

# 清理缓存
./yii order-config/clear
```

#### 2. 编程方式配置

```php
// 设置客户专属配置
$customerConfig = [
    'enabled' => true,
    'min_completion_rate' => 0.2,
    'max_partial_times' => 8,
    'require_approval' => false,
    'notification_enabled' => true,
    'priority_products' => ['HOT001', 'HOT002'],
    'blacklist_products' => ['FRAGILE001']
];

$result = $orderService->setPartialShipmentConfig($customerConfig, 'CUSTOMER_001');

// 获取生效配置
$config = $orderService->getPartialShipmentConfig('CUSTOMER_001', 'WH_001', 'NORMAL');
```

#### 3. 配置优先级示例

```php
// 场景1：普通客户 + 普通仓库 + 普通订单
$config1 = $orderService->getPartialShipmentConfig('NORMAL_CUSTOMER', 'NORMAL_WH', 'NORMAL');
// 结果：使用全局配置

// 场景2：VIP客户 + 普通仓库 + 普通订单  
$config2 = $orderService->getPartialShipmentConfig('VIP_CUSTOMER_001', 'NORMAL_WH', 'NORMAL');
// 结果：使用VIP客户专属配置（优先级最高）

// 场景3：普通客户 + 海外仓 + 普通订单
$config3 = $orderService->getPartialShipmentConfig('NORMAL_CUSTOMER', 'OVERSEAS_WH', 'NORMAL');
// 结果：使用海外仓专属配置

// 场景4：普通客户 + 普通仓库 + 紧急订单
$config4 = $orderService->getPartialShipmentConfig('NORMAL_CUSTOMER', 'NORMAL_WH', 'URGENT');
// 结果：使用紧急订单类型配置
```

---

## 性能监控

### 📊 监控指标

#### 1. 核心性能指标
- **查询响应时间**：缓存命中 <50ms，数据库查询 <200ms
- **缓存命中率**：目标 >95%，当前 ~85%
- **分布式锁等待时间**：平均 <10ms
- **订单处理吞吐量**：>1000 订单/分钟

#### 2. 业务指标监控
- **部分发货成功率**：目标 >98%
- **发货准确率**：目标 >99.9%
- **库存同步延迟**：<1秒
- **异常订单比例**：<1%

#### 3. Redis性能监控
```php
// Redis状态监控示例
$redis = Yii::$app->redis;

// 统计各类键的数量
$orderCacheCount = count($redis->keys('order:cache:*'));
$configCount = count($redis->keys('order:config:*'));
$lockCount = count($redis->keys('order:lock:*'));

echo "Redis性能统计：\n";
echo "- 订单缓存数量：{$orderCacheCount}\n";
echo "- 配置项数量：{$configCount}\n";
echo "- 活跃锁数量：{$lockCount}\n";

// 内存使用情况
$info = $redis->info('memory');
echo "- 内存使用：{$info['used_memory_human']}\n";
```

#### 4. 事件监控
```php
// 事件处理监控
$redis->subscribe(['order:events'], function($redis, $channel, $message) {
    $event = json_decode($message, true);
    
    // 记录事件处理时间
    $processingTime = microtime(true) - $event['timestamp'];
    
    // 统计事件类型分布
    $eventType = $event['type'];
    
    echo "事件处理：{$eventType}，耗时：{$processingTime}ms\n";
});
```

---

## 最佳实践

### 🎯 开发最佳实践

#### 1. 错误处理
```php
try {
    $result = $orderService->shipOrderStock($orderNo, $shipItems, $remark);
    
    if ($result['success']) {
        // 记录成功日志
        Yii::info("订单 {$orderNo} 发货成功", 'order-shipment');
    }
    
} catch (Exception $e) {
    // 记录错误日志
    Yii::error("订单 {$orderNo} 发货失败：" . $e->getMessage(), 'order-shipment');
    
    // 发送错误通知
    $this->notifyError($orderNo, $e->getMessage());
    
    throw $e;
}
```

#### 2. 缓存使用
```php
// 优先使用缓存
$order = $this->getOrderFromCache($orderNo);
if (!$order) {
    $order = $this->getOrderFromDatabase($orderNo);
    if ($order) {
        $this->setOrderCache($orderNo, $order);
    }
}

// 更新时清除缓存
$this->updateOrderStatus($orderNo, $newStatus);
$this->clearOrderCache($orderNo);
```

#### 3. 分布式锁使用
```php
// 获取锁
$lockKey = 'process_order:' . $orderNo;
$lockValue = $this->acquireLock($lockKey);

if (!$lockValue) {
    throw new Exception("订单正在处理中");
}

try {
    // 执行业务逻辑
    $this->processOrder($orderNo);
    
} finally {
    // 确保释放锁
    $this->releaseLock($lockKey, $lockValue);
}
```

#### 4. 配置管理
```php
// 获取配置时使用默认值
$config = $this->getPartialShipmentConfig($customer, $warehouse, $orderType);

// 使用配置时进行验证
if ($config['enabled'] && $completionRate >= $config['min_completion_rate']) {
    // 允许部分发货
    $this->executePartialShipment($orderNo, $shipItems);
} else {
    // 需要完整发货
    throw new Exception("不满足部分发货条件");
}
```

### 🔧 运维最佳实践

#### 1. 监控设置
- 设置关键指标告警阈值
- 监控Redis内存使用情况
- 跟踪分布式锁争用情况
- 定期检查缓存命中率

#### 2. 性能优化
- 定期清理过期缓存
- 优化热点数据缓存策略
- 监控慢查询并优化
- 调整Redis配置参数

#### 3. 故障处理
- 制定Redis故障应急预案
- 准备数据库降级方案
- 建立订单处理异常恢复流程
- 定期备份关键配置数据

#### 4. 容量规划
- 预估订单增长趋势
- 计算Redis内存需求
- 规划数据库扩容时机
- 评估网络带宽需求

---

## 相关文档

- **[Redis配置示例](redis-order-config-examples.php)** - 完整的Redis功能演示
- **[CLI管理工具](../console/controllers/OrderConfigController.php)** - 命令行配置管理
- **[系统优化建议](system-optimization-suggestions.md)** - 未来优化方向
- **[库存操作指南](inventory-operations-guide.md)** - 库存管理功能说明

---

*本文档持续更新，如有问题请提交Issue或联系开发团队。*
