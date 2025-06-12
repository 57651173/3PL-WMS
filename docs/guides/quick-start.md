# 快速开始指南

本文档提供海外仓库存管理系统的详细安装配置和基础使用示例。

## 📋 目录

1. [环境要求](#环境要求)
2. [安装配置](#安装配置)
3. [基础使用](#基础使用)
4. [配置说明](#配置说明)
5. [故障排除](#故障排除)

---

## 环境要求

### 系统环境
- **PHP**: 7.4+ (推荐 8.0+)
- **MySQL**: 5.7+ (推荐 8.0+)
- **Redis**: 5.0+ (推荐 6.2+)
- **Composer**: 2.0+
- **Web服务器**: Apache 2.4+ 或 Nginx 1.16+

### PHP扩展要求
```bash
# 必需扩展
php-mysql
php-redis
php-json
php-mbstring
php-openssl
php-tokenizer
php-xml
php-ctype
php-fileinfo

# 推荐扩展
php-opcache
php-gd
php-curl
```

### 硬件建议
- **CPU**: 2核心以上
- **内存**: 4GB以上（推荐8GB）
- **存储**: SSD硬盘，至少50GB可用空间
- **网络**: 稳定的网络连接

---

## 安装配置

### 🔧 1. 项目安装

#### 克隆项目
```bash
git clone <repository-url>
cd warehouse-server
```

#### 安装PHP依赖
```bash
# 使用Composer安装依赖
composer install

# 如果在生产环境，使用优化安装
composer install --no-dev --optimize-autoloader
```

#### 设置目录权限
```bash
# Linux/macOS环境
chmod -R 755 runtime/
chmod -R 755 web/assets/
chown -R www-data:www-data runtime/
chown -R www-data:www-data web/assets/

# 如果使用文件缓存
chmod -R 755 runtime/cache/
```

### 🗄️ 2. 数据库配置

#### 创建数据库
```sql
-- 创建数据库
CREATE DATABASE warehouse_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- 创建用户（可选）
CREATE USER 'warehouse_user'@'localhost' IDENTIFIED BY 'your_secure_password';
GRANT ALL PRIVILEGES ON warehouse_db.* TO 'warehouse_user'@'localhost';
FLUSH PRIVILEGES;
```

#### 配置数据库连接
```php
// environments/dev/common/config/main-local.php
<?php
return [
    'components' => [
        'db' => [
            'class' => 'yii\db\Connection',
            'dsn' => 'mysql:host=localhost;dbname=warehouse_db',
            'username' => 'warehouse_user',
            'password' => 'your_secure_password',
            'charset' => 'utf8mb4',
            'enableSchemaCache' => true,
            'schemaCacheDuration' => 3600,
            'schemaCache' => 'cache',
        ],
    ],
];
```

#### 执行数据库迁移
```bash
# 运行数据库迁移
./yii migrate

# 如果需要强制执行
./yii migrate --interactive=0

# 导入触发器（可选，用于自动维护库存一致性）
mysql -u warehouse_user -p warehouse_db < docs/stock_trigger.sql
```

### 🔴 3. Redis配置

#### Redis基础配置
```bash
# Ubuntu/Debian
sudo apt update
sudo apt install redis-server

# CentOS/RHEL
sudo yum install redis
# 或者使用dnf
sudo dnf install redis

# 启动Redis服务
sudo systemctl start redis
sudo systemctl enable redis

# 验证Redis运行状态
redis-cli ping
# 应该返回 PONG
```

#### 应用Redis配置
```php
// environments/dev/common/config/main-local.php
<?php
return [
    'components' => [
        'redis' => [
            'class' => 'yii\redis\Connection',
            'hostname' => 'localhost',
            'port' => 6379,
            'database' => 0,
            'password' => null,              // 如果Redis设置了密码
            'connectionTimeout' => 1,
            'dataTimeout' => 30,
            'retries' => 1,
        ],
        'cache' => [
            'class' => 'yii\redis\Cache',
            'redis' => 'redis',
            'keyPrefix' => 'warehouse_cache_',
            'defaultDuration' => 3600,       // 1小时缓存
        ],
    ],
];
```

#### Redis性能优化配置
```bash
# 编辑Redis配置文件
sudo nano /etc/redis/redis.conf

# 关键配置项
maxmemory 2gb
maxmemory-policy allkeys-lru
save 900 1
save 300 10
save 60 10000

# 重启Redis使配置生效
sudo systemctl restart redis
```

### 🌐 4. Web服务器配置

#### Nginx配置示例
```nginx
server {
    listen 80;
    server_name warehouse.local;
    root /var/www/warehouse-server/web;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(ht|svn|git) {
        deny all;
    }
}
```

#### Apache配置示例
```apache
<VirtualHost *:80>
    ServerName warehouse.local
    DocumentRoot /var/www/warehouse-server/web
    
    <Directory /var/www/warehouse-server/web>
        AllowOverride All
        Require all granted
    </Directory>
    
    ErrorLog ${APACHE_LOG_DIR}/warehouse_error.log
    CustomLog ${APACHE_LOG_DIR}/warehouse_access.log combined
</VirtualHost>
```

---

## 基础使用

### 📦 库存管理

#### 初始化服务
```php
<?php
use common\services\InventoryService;

// 创建库存服务实例
$inventoryService = new InventoryService();
```

#### 查询库存状态
```php
// 查询单个商品库存（带缓存优化）
$stock = $inventoryService->getStockStatusWithCache(
    'WH001',        // 仓库编码
    'CUSTOMER001',  // 客户编码
    'PRODUCT001',   // 商品SKU
    'BAR001'        // 商品条码
);

// 显示库存信息
echo "=== 库存状态 ===\n";
echo "商品SKU: {$stock['product_sku']}\n";
echo "可用库存: {$stock['stock_available']} 个\n";
echo "在途库存: {$stock['stock_onway']} 个\n";
echo "冻结库存: {$stock['stock_freeze']} 个\n";
echo "损坏库存: {$stock['stock_damaged']} 个\n";

// 批量查询多个商品
$products = ['PRODUCT001', 'PRODUCT002', 'PRODUCT003'];
$batchStock = $inventoryService->getBatchStockStatus(
    'WH001',
    'CUSTOMER001',
    $products
);

foreach ($batchStock as $productSku => $stock) {
    echo "{$productSku}: 可用库存 {$stock['stock_available']} 个\n";
}
```

#### 入库操作示例
```php
// 准备入库商品明细
$inboundItems = [
    [
        'product_sku' => 'PRODUCT001',
        'sku_barcode' => 'BAR001',
        'qty_expected' => 100,              // 预期数量
        'batch_no' => 'BATCH20231201',      // 批次号
        'production_date' => '2023-12-01',  // 生产日期
        'expiry_date' => '2024-12-01',      // 过期日期
        'location_code' => 'A001',          // 指定库位
        'supplier_code' => 'SUP001'         // 供应商编码
    ],
    [
        'product_sku' => 'PRODUCT002',
        'sku_barcode' => 'BAR002',
        'qty_expected' => 200,
        'batch_no' => 'BATCH20231202',
        'production_date' => '2023-12-02',
        'expiry_date' => '2024-12-02',
        'location_code' => 'B002',
        'supplier_code' => 'SUP001'
    ]
];

// 1. 提交入库单（增加在途库存）
$result = $inventoryService->addOnwayStock(
    'INB20231201001',           // 入库单号
    'WH001',                    // 仓库编码
    'CUSTOMER001',              // 客户编码
    $inboundItems,              // 商品明细
    '新商品入库'                 // 备注信息
);

if ($result) {
    echo "✓ 入库单提交成功，在途库存已增加\n";
} else {
    echo "✗ 入库单提交失败\n";
}

// 2. 收货上架（在途转可用）
$receiveItems = [
    [
        'product_sku' => 'PRODUCT001',
        'sku_barcode' => 'BAR001',
        'qty_received' => 95,               // 实际收货数量
        'location_code' => 'A001',
        'batch_no' => 'BATCH20231201'
    ],
    [
        'product_sku' => 'PRODUCT002',
        'sku_barcode' => 'BAR002',
        'qty_received' => 200,
        'location_code' => 'B002',
        'batch_no' => 'BATCH20231202'
    ]
];

$receiveResult = $inventoryService->receiveInboundStock(
    'INB20231201001',
    'WH001',
    'CUSTOMER001',
    $receiveItems,
    '收货完成，PRODUCT001短缺5个'
);

if ($receiveResult) {
    echo "✓ 收货上架成功，可用库存已更新\n";
}
```

#### 库存调整示例
```php
// 盘点调整
$adjustItems = [
    [
        'product_sku' => 'PRODUCT001',
        'sku_barcode' => 'BAR001',
        'location_code' => 'A001',
        'batch_no' => 'BATCH20231201',
        'qty_adjustment' => -3,             // 负数表示盘亏
        'reason' => '盘点发现缺失3个'
    ],
    [
        'product_sku' => 'PRODUCT002',
        'sku_barcode' => 'BAR002',
        'location_code' => 'B002',
        'batch_no' => 'BATCH20231202',
        'qty_adjustment' => 5,              // 正数表示盘盈
        'reason' => '盘点发现多出5个'
    ]
];

$adjustResult = $inventoryService->adjustStock(
    'ADJ20231201001',               // 调整单号
    'WH001',
    'CUSTOMER001',
    $adjustItems,
    '月度盘点调整'
);

if ($adjustResult) {
    echo "✓ 库存调整完成\n";
}
```

### 🚚 订单处理

#### 初始化订单服务
```php
<?php
use common\services\OrderService;

// 创建订单服务实例
$orderService = new OrderService();
```

#### 预拣货排序
```php
// 执行预拣货排序（智能路径优化）
$orderNo = 'XS-ORDER-20231201-001';
$sortingResult = $orderService->performPrePickSorting($orderNo);

if ($sortingResult) {
    echo "=== 预拣货排序结果 ===\n";
    echo "订单号: {$orderNo}\n";
    echo "商品种类: {$sortingResult['total_items']}\n";
    echo "涉及库位: " . count($sortingResult['picking_locations']) . " 个\n";
    echo "库存充足商品: {$sortingResult['stock_check']['sufficient_stock']}\n";
    echo "库存不足商品: {$sortingResult['stock_check']['insufficient_stock']}\n";
    
    // 显示拣货路径
    echo "\n=== 推荐拣货路径 ===\n";
    foreach ($sortingResult['picking_locations'] as $index => $location) {
        echo "第" . ($index + 1) . "站: 库位 {$location['location_code']}\n";
        foreach ($location['items'] as $item) {
            echo "  - {$item['product_sku']}: 拣货 {$item['pick_quantity']} 个\n";
        }
    }
}
```

#### 订单发货处理
```php
// 完整发货（发货所有商品）
$shipResult = $orderService->shipOrderStock(
    $orderNo,
    [],                                 // 空数组表示发货全部商品
    '正常发货'
);

if ($shipResult['success']) {
    echo "=== 发货结果 ===\n";
    echo "发货类型: {$shipResult['shipment_type']}\n";
    echo "发货商品数: {$shipResult['shipped_items']}\n";
    echo "订单状态: {$shipResult['order_status']}\n";
    echo "完成率: {$shipResult['shipment_summary']['completion_rate']}%\n";
    
    if ($shipResult['shipment_type'] === 'complete') {
        echo "✓ 订单已完全发货\n";
    } else {
        echo "⚠ 部分发货，还有商品未发\n";
    }
}

// 部分发货（指定具体商品和数量）
$partialOrderNo = 'XS-ORDER-20231201-002';
$partialShipItems = [
    [
        'product_sku' => 'PRODUCT001',
        'sku_barcode' => 'BAR001',
        'qty_shipped' => 50,              // 仅发货50个
        'batch_no' => 'BATCH20231201',
        'location_code' => 'A001'
    ]
];

$partialResult = $orderService->shipOrderStock(
    $partialOrderNo,
    $partialShipItems,
    '第一批部分发货'
);

if ($partialResult['success'] && $partialResult['shipment_type'] === 'partial') {
    echo "=== 部分发货结果 ===\n";
    echo "本次发货商品: {$partialResult['shipped_items']}\n";
    echo "剩余待发货: {$partialResult['remaining_items']}\n";
    echo "当前完成率: {$partialResult['shipment_summary']['completion_rate']}%\n";
}
```

#### 订单取消和退货
```php
// 订单取消（释放冻结库存）
$cancelOrderNo = 'XS-ORDER-20231201-003';
$cancelResult = $orderService->cancelOrderStock(
    $cancelOrderNo,
    '客户主动取消订单'
);

if ($cancelResult) {
    echo "✓ 订单 {$cancelOrderNo} 取消成功，冻结库存已释放\n";
}

// 退货处理（商品重新入库）
$returnItems = [
    [
        'product_sku' => 'PRODUCT001',
        'sku_barcode' => 'BAR001',
        'qty_returned' => 10,
        'location_code' => 'RETURN01',      // 退货专用库位
        'batch_no' => 'BATCH20231201',
        'reason' => '客户不满意',
        'return_condition' => 'good'        // 商品状况良好
    ]
];

$returnResult = $orderService->returnOrderAddStock(
    'XS-ORDER-20231201-004',
    $returnItems,
    '正常退货重新入库'
);

if ($returnResult) {
    echo "✓ 退货处理成功，商品已重新入库\n";
}
```

#### 获取订单详细信息
```php
// 获取订单完整信息
$orderDetails = $orderService->getOrderFullDetails($orderNo);

if ($orderDetails) {
    echo "=== 订单详细信息 ===\n";
    echo "订单号: {$orderDetails['order']['order_no']}\n";
    echo "订单状态: {$orderDetails['order']['order_status']}\n";
    echo "客户编码: {$orderDetails['order']['customer_code']}\n";
    echo "仓库编码: {$orderDetails['order']['warehouse_code']}\n";
    
    echo "\n=== 库存状态检查 ===\n";
    echo "商品种类总数: {$orderDetails['stock_status']['total_products']}\n";
    echo "库存充足商品: {$orderDetails['stock_status']['sufficient_stock']}\n";
    echo "库存不足商品: {$orderDetails['stock_status']['insufficient_stock']}\n";
    
    // 显示每个商品的库存状态
    echo "\n=== 商品库存详情 ===\n";
    foreach ($orderDetails['stock_status']['stock_details'] as $detail) {
        $status = $detail['is_sufficient'] ? '✓ 充足' : '✗ 不足';
        echo "{$status} {$detail['product_sku']}: ";
        echo "需要 {$detail['qty_ordered']} 个, ";
        echo "可用 {$detail['qty_available']} 个\n";
    }
}
```

---

## 配置说明

### 🔧 Redis配置优化

#### 分布式锁配置
```php
// common/config/main.php
'components' => [
    'redis' => [
        'class' => 'yii\redis\Connection',
        'hostname' => 'localhost',
        'port' => 6379,
        'database' => 0,
        'connectionTimeout' => 1,
        'dataTimeout' => 30,
        'retries' => 1,
    ],
];
```

#### 缓存配置
```php
'cache' => [
    'class' => 'yii\redis\Cache',
    'redis' => 'redis',
    'keyPrefix' => 'warehouse_',
    'defaultDuration' => 3600,
    'serializer' => false,          // 如果只存储字符串
];
```

### 🗄️ 数据库连接池配置

```php
'db' => [
    'class' => 'yii\db\Connection',
    'dsn' => 'mysql:host=localhost;dbname=warehouse_db',
    'username' => 'warehouse_user',
    'password' => 'your_password',
    'charset' => 'utf8mb4',
    'enableSchemaCache' => true,
    'schemaCacheDuration' => 3600,
    'schemaCache' => 'cache',
    'attributes' => [
        PDO::ATTR_TIMEOUT => 30,
        PDO::MYSQL_ATTR_INIT_COMMAND => "SET sql_mode='STRICT_TRANS_TABLES'",
    ],
];
```

### 📊 日志配置

```php
'log' => [
    'traceLevel' => YII_DEBUG ? 3 : 0,
    'targets' => [
        [
            'class' => 'yii\log\FileTarget',
            'levels' => ['error', 'warning'],
            'logFile' => '@runtime/logs/app.log',
            'maxFileSize' => 10240,     // 10MB
            'maxLogFiles' => 5,
        ],
        [
            'class' => 'yii\log\FileTarget',
            'levels' => ['info'],
            'categories' => ['inventory-operation', 'order-operation'],
            'logFile' => '@runtime/logs/business.log',
            'maxFileSize' => 10240,
            'maxLogFiles' => 10,
        ],
    ],
];
```

---

## 故障排除

### 🔧 常见问题

#### 1. Redis连接失败
```bash
# 检查Redis服务状态
sudo systemctl status redis

# 检查Redis端口
netstat -tlnp | grep :6379

# 测试Redis连接
redis-cli ping

# 查看Redis日志
sudo tail -f /var/log/redis/redis-server.log
```

#### 2. 数据库连接问题
```bash
# 检查MySQL服务
sudo systemctl status mysql

# 测试数据库连接
mysql -u warehouse_user -p -h localhost warehouse_db

# 检查MySQL错误日志
sudo tail -f /var/log/mysql/error.log
```

#### 3. 权限问题
```bash
# 检查目录权限
ls -la runtime/
ls -la web/assets/

# 重新设置权限
sudo chown -R www-data:www-data runtime/
sudo chmod -R 755 runtime/
```

#### 4. Composer依赖问题
```bash
# 清理Composer缓存
composer clear-cache

# 重新安装依赖
rm -rf vendor/
composer install

# 检查PHP扩展
php -m | grep -E "redis|mysql|json"
```

### 🐛 调试模式

#### 启用调试模式
```php
// environments/dev/common/config/main-local.php
defined('YII_DEBUG') or define('YII_DEBUG', true);
defined('YII_ENV') or define('YII_ENV', 'dev');
```

#### 查看系统日志
```bash
# 查看应用日志
tail -f runtime/logs/app.log

# 查看业务日志
tail -f runtime/logs/business.log

# 查看Web服务器日志
tail -f /var/log/nginx/error.log    # Nginx
tail -f /var/log/apache2/error.log  # Apache
```

### 📊 性能监控

#### 监控Redis性能
```bash
# Redis监控命令
redis-cli info stats
redis-cli info memory
redis-cli monitor

# 查看慢查询
redis-cli slowlog get 10
```

#### 监控MySQL性能
```sql
-- 查看当前连接
SHOW PROCESSLIST;

-- 查看慢查询
SHOW VARIABLES LIKE 'slow_query_log';
SHOW VARIABLES LIKE 'long_query_time';

-- 查看系统状态
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Queries';
```

---

## 下一步

### 📚 深入学习
- 阅读 **[库存操作完整指南](inventory-operations-guide.md)** 了解18种库存操作
- 查看 **[订单处理操作指南](order-operations-guide.md)** 掌握订单处理流程
- 参考 **[业务使用示例](business-examples.md)** 学习更多业务场景

### 🔧 系统优化
- 根据业务量调整Redis和MySQL配置
- 设置监控和告警系统
- 制定备份和灾难恢复计划

### 🚀 功能扩展
- 集成ERP系统
- 添加API接口
- 开发前端管理界面

---

*如有问题，请查看项目文档或提交GitHub Issue获取技术支持。* 