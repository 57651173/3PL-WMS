# 3PL-WMS Warehouse Management System

## 🌐 Language Versions / 语言版本
- [US English ](README.md) - Current Page
- [CN 中文](README_ZH.md)
- [DE Deutsch](README_DE.md)
- [JP 日本語 ](README_JA.md)

---

## 📋 Project Overview

This project is a core module of an enterprise-level warehouse management system, focusing on order processing, inventory management, picking operations, and shipping processes. The system adopts modern architectural design to support high-concurrency, high-availability warehouse operation scenarios and strictly follows the FIFO (First In, First Out) principle for inventory management.

## 🏗️ System Architecture

This system adopts an **architectural design that separates the data access layer from the business logic layer**, ensuring code maintainability, security, and scalability.

### Architectural Advantages
- **🧩 Clear Responsibilities**: Separation of data layer and business layer
- **🚀 Easy Maintenance**: Clear code structure, easy to extend
- **⚡ Development Efficiency**: Reduce repetitive work, improve collaboration efficiency

### Layered Architecture Design

```
Business Application Layer (Services)
          ↓
Business Logic Layer (models)
          ↓
Data Access Layer (tables)
          ↓
Database Layer (MySQL Tables)
```

### Core Architectural Principles

1. **Database Table Structure Change Safety**
   - When database table structures change, only need to use Gii to regenerate table classes in `common/tables`
   - Business logic code in `common/models` remains completely unaffected, ensuring code safety

2. **Centralized Business Logic Management**
   - All business-related methods, computed properties, and state management are implemented in the `models` layer
   - Avoid scattering business logic in different places, facilitating maintenance and testing

3. **Team Collaboration Friendly**
   - New members can easily understand the architectural layering and get started quickly
   - Reduce code conflicts caused by Gii regeneration
   - Clear division of responsibilities, improving development efficiency

### 📖 Documentation Resources

For detailed technical documentation, please visit: **[📖 Documentation Center](docs/README.md)**

### 📖 Quick Navigation

| Category | Description | Link |
|----------|-------------|------|
| 🏗️ **Architecture Design** | System architecture, development standards and best practices | [View Docs](docs/architecture/README.md) |
| 📋 **User Guides** | Quick start, operation guides and process instructions | [View Docs](docs/guides/) |
| 🔧 **Technical Guides** | Concurrency control, performance optimization and architecture refactoring | [View Docs](docs/technical/) |
| 💡 **Code Examples** | Business scenario examples and code references | [View Docs](docs/examples/) |
| 📊 **Database Design** | Table structure design and data model documentation | [View Docs](docs/database/) |
| 🔌 **API Documentation** | Interface documentation and usage examples | [View Docs](docs/api/) |

> 💡 **Tip**: All documentation is written in respective languages with complete code examples and best practice guidance.

---

## 🔄 Complete Business Process

### Phase 1: Order Processing and Picking Detail Generation

```
1. Order Submission (STATUS_PEND)
   ↓
2. Order Approval (STATUS_CONF)
   ↓
3. Pre-picking Sorting + FIFO Inventory Freeze (STATUS_PICK)
   ↓
4. Generate Picking Details (picking_no = null, status = pending)
```

**Key Operations:**
- Order information validation
- FIFO inventory pre-check
- Status flow control

### Phase 2: Wave Picking and Execution

```
5. Picking rules batch binding → Supplement picking order number → Picker picking → Packaging → Shipping
6. Support multiple wave strategies (by time, location, customer, priority, etc.)
7. Intelligent merging of same SKUs, optimize picking paths
8. Real-time picking progress tracking
```

**Core Features:**
- **FIFO Inventory Freeze** - Freeze batch inventory based on first-in-first-out principle by inbound time
- **Intelligent Path Optimization** - Optimize picking paths by sorting location codes
- **Multi-dimensional Inventory Management** - Three-level inventory statistics: batch, location, warehouse
- **Wave Picking** - Assign picking order numbers based on wave rules

### Phase 3: Shipping Processing

```
9. Automatic Shipping Processing (STATUS_SHIP)
    ↓
10. FIFO Reduce Frozen Inventory
    ↓
11. Logistics Handover
    ↓
12. Customer Delivery (STATUS_FINISH)
```

## 🛠️ Technical Features

### FIFO Inventory Management (Core Feature)

#### 1. FIFO Principle Implementation

```php
// All inventory operations are sorted by inbound time, ensuring first-in-first-out
->orderBy('received_at ASC, created_at ASC')
```

#### 2. FIFO Inventory Freeze

- **Auto Allocation Freeze** - `autoAllocateStockFreeze()` automatically selects batches for freezing based on FIFO principle
- **Specified Location Freeze** - `freezeLocationStockByFifo()` freezes within specified locations based on FIFO principle
- **Batch Traceability** - Complete recording of each batch's freeze status and inbound time

```php
// Example: FIFO Inventory Freeze
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
    "Order inventory freeze"
);
```

#### 3. FIFO Inventory Release

- **Auto Release** - `autoReleaseStockFreeze()` releases frozen inventory based on FIFO principle
- **Batch Matching** - Prioritize releasing the earliest frozen batches
- **Inventory Balance** - Automatically update three-level inventory statistics: batch, location, warehouse

#### 4. FIFO Inventory Deduction

- **Auto Deduction** - `autoDeductStockFromLocation()` deducts available inventory based on FIFO principle
- **Shipping Deduction** - Prioritize deducting the earliest inbound batches during shipping
- **Cost Accounting** - Support FIFO cost accounting method

#### 5. FIFO Advantages

- **Inventory Turnover Optimization** - Ensure earlier inbound goods are used first, avoiding inventory accumulation
- **Shelf Life Management** - For goods with shelf life, FIFO can reduce expiration risks
- **Accurate Cost Accounting** - Use inventory in inbound order for more accurate cost accounting
- **Inventory Traceability** - Complete batch traceability chain for quality issue investigation
- **Compliance Requirements** - Meet FIFO compliance requirements in certain industries

#### 6. Applicable Scenarios

- **Food & Beverage** - Strict shelf life management
- **Pharmaceutical Industry** - Drug batch management and expiration date control
- **Chemical Products** - Chemical stability and safety management
- **Electronic Products** - Avoid component aging and technical obsolescence
- **Textile Industry** - Inventory turnover for seasonal goods

### Redis Optimization

- **Distributed Lock** - Prevent concurrent operation conflicts
- **Caching Mechanism** - Improve query performance
- **Event Publishing** - Real-time status notifications
- **Progress Tracking** - Real-time picking progress

#### Concurrency Control Mechanism

The system uses Redis distributed locks to ensure data consistency in high-concurrency scenarios:

```php
// Batch inventory lock - finest granularity
$lockKey = "batch_stock_lock:{$warehouseCode}:{$customerCode}:{$productSku}:{$skuBarcode}:{$lotNumber}";

// Location inventory lock - medium granularity
$lockKey = "location_stock_lock:{$warehouseCode}:{$locationCode}:{$customerCode}:{$productSku}:{$skuBarcode}";

// Warehouse inventory lock - coarse granularity
$lockKey = "warehouse_stock_lock:{$warehouseCode}:{$customerCode}:{$productSku}:{$skuBarcode}";
```

**Lock Characteristics:**
- **Lock Timeout**: 30 seconds, prevent deadlocks
- **Max Wait Time**: 10 seconds, avoid long blocking
- **Atomic Operations**: Ensure atomicity of inventory updates
- **Negative Inventory Check**: Prevent overselling
- **Auto Retry**: Automatic retry for temporary errors

**Concurrency Safety Guarantees:**
- Multiple orders freeze inventory simultaneously, executed in sequence
- Data consistency maintained when inbound and outbound operations are concurrent
- Prevent data race during batch inventory updates
- Real-time accurate inventory statistics, no data loss

### Intelligent Path Optimization

```php
// Sort by location code to optimize picking paths
usort($details, function($a, $b) {
    return strcmp($a['location_code'], $b['location_code']);
});
```

## 📊 Usage Examples

### 1. FIFO Inventory Freeze Example

```php
$inventoryService = new InventoryService();

// View current inventory status (sorted by FIFO)
$stockStatus = $inventoryService->getDetailedStockStatus('WH01', 'CUST001', 'SKU001', 'BAR001');

// Execute FIFO inventory freeze
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
    "FIFO inventory freeze demonstration"
);

// System automatically selects batches for freezing based on first-in-first-out principle by inbound time
```

### 2. Order Processing Phase

```php
$orderService = new OrderService();
$result = $orderService->performPrePickSorting('SO202412150001', true);

// Return results include:
// - Generate picking details (following FIFO principle)
```

### 3. Wave Picking Phase

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

// Return results include:
// - Generate picking order
```

### 4. Manual Detail Assignment

```php
$waveService = new WavePickingService();
$assignResult = $waveService->assignDetailsToPickingManually(
    [1, 2, 3], // Detail ID array
    'WWH0120241215000001' // Picking order number
);

// Return results include:
// - Number of successfully assigned details
```

### 5. Picking Execution Phase

```php
$pickingService = new PickingService();
$pickingService->startPicking('WWH0120241215000001', 'picker001');
$pickingService->scanAndPick('WWH0120241215000001', 'SKU001', 10, 'BATCH001');
$pickingService->completePicking('WWH0120241215000001');
```

## 🔧 Configuration Options

### Partial Shipment Configuration

```php
$config = [
    'enabled' => true,                    // Enable partial shipment
    'min_completion_rate' => 0.8,         // Minimum completion rate 80%
    'max_partial_times' => 3,             // Maximum partial shipment times
    'auto_ship_threshold' => 0.9,         // Auto shipment threshold 90%
    'require_approval' => false,          // No approval required
    'notification_enabled' => true,       // Enable notifications
];

$orderService->setPartialShipmentConfig($config, 'CUSTOMER001', 'WH001');
```

### Picking Type Configuration

```php
$pickingOptions = [
    'picking_type' => PickingService::TYPE_NORMAL,  // Normal picking
    'picker' => 'picker001',                        // Assigned picker
    'priority' => 1                                 // Priority
];
```

## 📈 Performance Optimization

### 1. Redis Caching Strategy

- **Order Information Cache** - 5 minutes TTL
- **Picking Progress Cache** - 30 minutes TTL
- **Configuration Information Cache** - 1 hour TTL
- **FIFO Batch Cache** - 10 minutes TTL

### 2. Database Optimization

- **Index Optimization** - Build composite indexes for key fields
- **FIFO Index** - `(warehouse_code, customer_code, product_sku, received_at, created_at)`
- **Batch Operations** - Reduce database round trips
- **Paginated Queries** - Paginated processing for large data volumes

### 3. Concurrency Control

- **Distributed Lock** - Prevent concurrent conflicts
- **Atomic Operations** - Ensure data consistency
- **Retry Mechanism** - Handle temporary errors

## 🚀 Deployment Instructions

### Environment Requirements

- PHP 7.4+
- Yii2 Framework
- MySQL 5.7+
- Redis 5.0+

## 🎯 Future Roadmap

1. **AI Path Optimization** - Machine learning to optimize picking paths
2. **Automation Integration** - Integration with automated equipment
3. **Mobile Support** - Develop mobile picking applications
4. **Data Analytics** - Picking efficiency analysis reports
5. **Multi-warehouse Support** - Cross-warehouse transfer functionality

## 📞 Technical Support

For questions or suggestions, please contact the development team.

<img src="./docs/assets/images/image/wechat.png" alt="donate" width="200" />

## 📚 Documentation Resources

For detailed technical documentation, please visit: **[📖 Documentation Center](docs/README.md)**

### 📖 Quick Navigation

| Category | Description | Link |
|----------|-------------|------|
| 🏗️ **Architecture Design** | System architecture, development standards and best practices | [View Docs](docs/architecture/README.md) |
| 📋 **User Guides** | Quick start, operation guides and process instructions | [View Docs](docs/guides/) |
| 🔧 **Technical Guides** | Concurrency control, performance optimization and architecture refactoring | [View Docs](docs/technical/) |
| 💡 **Code Examples** | Business scenario examples and code references | [View Docs](docs/examples/) |
| 📊 **Database Design** | Table structure design and data model documentation | [View Docs](docs/database/) |
| 🔌 **API Documentation** | Interface documentation and usage examples | [View Docs](docs/api/) |

--- 