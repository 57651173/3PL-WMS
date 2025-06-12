# 系统架构设计文档

## 📚 目录
- [核心架构理念](#核心架构理念)
- [分层架构设计](#分层架构设计)
- [目录结构规范](#目录结构规范)
- [开发规范](#开发规范)
- [最佳实践](#最佳实践)
- [代码示例](#代码示例)
- [常见问题](#常见问题)

---

## 🎯 核心架构理念

### 设计原则

本系统采用**数据访问层与业务逻辑层分离**的架构设计，确保代码的可维护性、安全性和扩展性。

#### 核心思想
- **`common/tables`** - 数据访问层（DAL），存放 Yii Gii 自动生成的基础表类
- **`common/models`** - 业务逻辑层（BLL），存放继承表类的业务模型
- **`common/service`** - 业务应用层

#### 设计优势

1. **🔒 代码安全性**
   - Gii 重新生成不影响业务代码
   - 数据表结构变更时，只需重新生成 `tables` 中的类
   - `models` 中的业务逻辑完全安全

2. **🧩 关注点分离**
   - Tables: 专注于数据结构映射
   - Models: 专注于业务逻辑实现

3. **🚀 维护性和扩展性**
   - 业务逻辑集中管理
   - 代码结构清晰，职责明确
   - 便于团队协作和代码审查

4. **⚡ 开发效率**
   - 避免因 Gii 重新生成导致的代码冲突
   - 新团队成员容易理解架构
   - 减少重复开发工作

### 📋 实际代码对比示例

#### 传统方式 vs 分层架构方式

**传统做法问题：**
```php
// ❌ 传统做法：直接在Gii生成的类中添加业务逻辑
class KySalesOrder extends \yii\db\ActiveRecord 
{
    // Gii生成的基础内容
    public static function tableName() { return 'ky_sales_order'; }
    public function rules() { /* 验证规则 */ }
    
    // 开发者添加的业务方法 - 危险！
    public function confirmOrder() {
        $this->order_status = 10;
        return $this->save();
    }
    
    public function getStatusLabel() {
        // 业务逻辑代码...
    }
    
    // 当数据表结构变化，重新使用Gii生成时
    // 上面的业务方法会被覆盖！！！
}
```

**分层架构做法：**

**数据访问层** (`common/tables/KySalesOrder.php`) - Gii 生成和维护
```php
<?php
namespace common\tables;

/**
 * 销售订单表类 - 由 Gii 自动生成
 * 
 * @property int $order_id
 * @property string $order_no
 * @property int $order_status
 * // ... 其他属性
 */
class KySalesOrder extends \yii\db\ActiveRecord
{
    public static function tableName()
    {
        return 'ky_sales_order';
    }

    public function rules()
    {
        return [
            [['order_no', 'customer_code'], 'required'],
            [['order_status'], 'integer'],
            // ... 基础验证规则
        ];
    }

    public function attributeLabels()
    {
        return [
            'order_id' => 'Order ID',
            'order_no' => 'Order No',
            // ... 基础标签
        ];
    }
    
    // ✅ 只包含数据结构相关的基础内容
    // ✅ 可以安全地被 Gii 重新生成而不影响业务代码
}
```

**业务逻辑层** (`common/models/SalesOrder.php`) - 手动编写和维护
```php
<?php
namespace common\models;

use common\tables\KySalesOrder;
use yii\helpers\ArrayHelper;

/**
 * 销售订单业务模型
 * 
 * 继承自 KySalesOrder，提供销售订单相关的业务逻辑
 */
class SalesOrder extends KySalesOrder
{
    use BaseModel;

    // 业务状态常量
    const STATUS_PEND = 1;       // 待审核
    const STATUS_CONF = 10;      // 已确认
    const STATUS_SHIP = 80;      // 已发货
    const STATUS_FINISH = 100;   // 已签收
    const STATUS_CANCEL = 0;     // 已取消

    /**
     * ✅ 业务逻辑方法 - 安全不会被覆盖
     */
    public function confirmOrder()
    {
        if (!$this->canConfirm()) {
            return false;
        }
        
        $this->order_status = self::STATUS_CONF;
        $this->updated_at = date('Y-m-d H:i:s');
        
        return $this->save();
    }

    /**
     * ✅ 状态判断方法 - 业务逻辑安全
     */
    public function canConfirm()
    {
        return $this->order_status === self::STATUS_PEND;
    }

    /**
     * ✅ 计算属性 - 不会丢失
     */
    public function getStatusLabel()
    {
        $labels = [
            self::STATUS_PEND => '待审核',
            self::STATUS_CONF => '已确认',
            self::STATUS_SHIP => '已发货',
            self::STATUS_FINISH => '已签收',
            self::STATUS_CANCEL => '已取消',
        ];
        
        return ArrayHelper::getValue($labels, $this->order_status, '未知状态');
    }

    /**
     * ✅ 查询作用域 - 复杂查询逻辑安全
     */
    public static function findPending()
    {
        return static::find()->where(['order_status' => self::STATUS_PEND]);
    }

    /**
     * ✅ 关联关系扩展 - 业务级别的关联
     */
    public function getPickingOrders()
    {
        return $this->hasMany(SalesPickingOrder::class, ['order_no' => 'order_no']);
    }
    
    // ... 更多业务方法
}
```

#### 架构优势对比

| 对比项 | 传统方式 | 分层架构方式 |
|-------|---------|-------------|
| **代码安全性** | ❌ Gii重新生成会覆盖业务代码 | ✅ 业务代码完全安全，永不丢失 |
| **维护性** | ❌ 数据结构和业务逻辑混在一起 | ✅ 职责清晰，易于维护 |
| **团队协作** | ❌ 容易产生冲突和误操作 | ✅ 分工明确，减少冲突 |
| **扩展性** | ❌ 难以扩展复杂业务逻辑 | ✅ 业务逻辑层可以自由扩展 |
| **测试友好** | ❌ 数据层和业务层耦合难测试 | ✅ 业务逻辑独立，便于单元测试 |
| **代码复用** | ❌ 业务逻辑散落，难以复用 | ✅ 业务模型可在不同场景复用 |

### 🔄 实际开发流程示例

#### 场景：数据表增加新字段

**第一步：数据库变更**
```sql
-- 为订单表添加新字段
ALTER TABLE ky_sales_order ADD COLUMN `priority_level` INT DEFAULT 5 COMMENT '优先级级别';
ALTER TABLE ky_sales_order ADD COLUMN `estimated_ship_date` DATE NULL COMMENT '预计发货日期';
```

**第二步：重新生成表类（Gii）**
```bash
# 使用 Gii 重新生成 KySalesOrder 类
# ✅ 新字段自动添加到 rules() 和 attributeLabels() 中
# ✅ 业务逻辑层的 SalesOrder 完全不受影响
```

**第三步：在业务层添加新功能**
```php
// common/models/SalesOrder.php
class SalesOrder extends KySalesOrder
{
    // 优先级常量
    const PRIORITY_URGENT = 1;    // 紧急
    const PRIORITY_HIGH = 3;      // 高优先级  
    const PRIORITY_NORMAL = 5;    // 普通
    const PRIORITY_LOW = 8;       // 低优先级

    /**
     * ✅ 新增：获取优先级标签
     */
    public function getPriorityLabel()
    {
        $labels = [
            self::PRIORITY_URGENT => '紧急',
            self::PRIORITY_HIGH => '高优先级',
            self::PRIORITY_NORMAL => '普通',
            self::PRIORITY_LOW => '低优先级',
        ];
        
        return ArrayHelper::getValue($labels, $this->priority_level, '普通');
    }

    /**
     * ✅ 新增：检查是否紧急订单
     */
    public function isUrgent()
    {
        return $this->priority_level === self::PRIORITY_URGENT;
    }

    /**
     * ✅ 新增：按优先级查询
     */
    public static function findByPriority($priority)
    {
        return static::find()->where(['priority_level' => $priority]);
    }

    /**
     * ✅ 新增：获取预计发货状态
     */
    public function getShipmentStatus()
    {
        if (!$this->estimated_ship_date) {
            return '未设置发货日期';
        }
        
        $today = date('Y-m-d');
        if ($this->estimated_ship_date < $today) {
            return '延期发货';
        } elseif ($this->estimated_ship_date == $today) {
            return '今日发货';
        } else {
            return '按时发货';
        }
    }
    
    // ✅ 原有的所有业务方法保持不变
    // confirmOrder(), getStatusLabel(), findPending() 等都正常工作
}
```

**第四步：无缝使用新功能**
```php
// 控制器或服务中的使用
$order = SalesOrder::findOne(['order_no' => 'ORD20241215001']);

// ✅ 原有功能正常
echo $order->getStatusLabel();  // "已确认"
$order->confirmOrder();         // 状态流转正常

// ✅ 新功能立即可用
echo $order->getPriorityLabel();    // "紧急"
echo $order->getShipmentStatus();   // "今日发货"

// ✅ 新的查询方法
$urgentOrders = SalesOrder::findByPriority(SalesOrder::PRIORITY_URGENT)->all();
```

### 📊 架构价值总结

通过这种分层架构设计，我们实现了：

1. **零风险数据结构变更**：可以随时使用 Gii 重新生成表类，业务代码永远安全
2. **业务逻辑集中管理**：所有订单相关的业务逻辑都在 `SalesOrder` 模型中
3. **渐进式功能增强**：可以随时添加新的业务方法而不影响现有功能
4. **团队协作友好**：数据层和业务层分工明确，减少代码冲突
5. **测试和维护便利**：业务逻辑独立，便于编写单元测试和维护

这种架构模式特别适合：
- 数据表结构经常变化的项目
- 需要频繁添加业务功能的系统
- 多人协作开发的团队项目
- 对代码安全性要求较高的企业应用

---

## 🏗️ 分层架构设计

### 架构图

```
┌─────────────────────────────────────────────────────────────┐
│                     业务应用层                                │
│                (Controllers/Services)                      │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                   业务逻辑层                                  │
│                 (common/models)                            │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│  │ SalesPicking │ │InventoryBatch│ │ SalesOrder  │          │
│  │              │ │              │ │             │          │
│  │ + 业务方法    │ │ + 计算属性    │ │ + 状态管理   │          │
│  │ + 状态管理    │ │ + 状态判断    │ │ + 关联扩展   │          │
│  │ + 行为扩展    │ │ + 业务验证    │ │ + 业务规则   │          │
│  └─────────────┘ └─────────────┘ └─────────────┘          │
│         │                │               │                 │
│         ▼                ▼               ▼                 │
└─────────────────────────────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                   数据访问层                                  │
│                 (common/tables)                            │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│  │KySalesPicking│ │KyInventoryBatch│ │KySalesOrder│          │
│  │              │ │              │ │             │          │
│  │ + tableName()│ │ + rules()    │ │ + relations │          │
│  │ + rules()    │ │ + attributes │ │ + validation│          │
│  │ + attributes │ │ + relations  │ │             │          │
│  └─────────────┘ └─────────────┘ └─────────────┘          │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                    数据库层                                   │
│                  (MySQL Tables)                            │
└─────────────────────────────────────────────────────────────┘
```

### 层级职责

#### 1. 数据访问层 (`common/tables`)
**职责**: 纯数据表映射，由 Gii 自动生成和维护

```php
/**
 * 数据访问层 - 仅包含数据结构定义
 * 
 * ✅ 包含内容:
 * - tableName()        表名定义
 * - rules()           数据验证规则
 * - attributeLabels()  字段标签
 * - 基础关联关系        hasOne/hasMany
 * 
 * ❌ 不包含内容:
 * - 业务逻辑方法
 * - 复杂计算属性
 * - 状态管理逻辑
 * - 自定义行为
 */
class KySalesPickingOrder extends \yii\db\ActiveRecord
{
    // Gii 生成的标准内容...
}
```

#### 2. 业务逻辑层 (`common/models`)
**职责**: 业务逻辑实现，继承并扩展数据访问层

```php
/**
 * 业务逻辑层 - 实现具体业务功能
 * 
 * ✅ 包含内容:
 * - 业务方法            计算、处理、转换
 * - 计算属性            getXxx() 方法
 * - 状态管理            状态判断、转换
 * - 业务验证            复杂验证规则
 * - 关联关系扩展         复杂查询、聚合
 * - 行为绑定            Behaviors
 * - 事件处理            beforeSave/afterSave
 * 
 * ❌ 避免内容:
 * - 重复基础 rules()
 * - 重复 tableName()
 * - 简单的 attributeLabels() 重写
 */
class SalesPickingOrder extends KySalesPickingOrder
{
    use BaseModel;
    
    // 业务逻辑实现...
}
```

---

## 📁 目录结构规范

### 标准目录结构

```
common/
├── tables/                     # 数据访问层 (Gii 生成)
│   ├── KySalesPickingOrder.php # 拣货订单表类
│   ├── KyInventoryBatch.php    # 库存批次表类
│   ├── KySalesOrder.php        # 销售订单表类
│   └── ...                     # 其他表类
│
├── models/                     # 业务逻辑层 (手动编写)
│   ├── SalesPickingOrder.php   # 拣货订单业务类
│   ├── InventoryBatch.php      # 库存批次业务类
│   ├── SalesOrder.php          # 销售订单业务类
│   ├── BaseModel.php           # 基础模型 trait
│   ├── BaseForm.php            # 基础表单类
│   └── ...                     # 其他业务类
│
├── services/                   # 服务层 (复杂业务逻辑)
│   ├── InventoryService.php    # 库存服务
│   ├── OrderService.php        # 订单服务
│   └── ...                     # 其他服务
│
└── components/                 # 组件层 (工具类)
    ├── RedisHelper.php         # Redis 助手
    ├── DatabaseHelper.php      # 数据库助手
    └── ...                     # 其他组件
```

### 命名规范

#### Tables 命名规范
- **保持 Gii 生成的表名前缀**: `KySalesPickingOrder`
- **使用 PascalCase**: 每个单词首字母大写
- **表前缀含义**: 
  - `Ky` - 业务核心表
  - `Sys` - 系统配置表
  - `Ka` - 基础数据表
  - `Oms` - 订单管理表

#### Models 命名规范
- **去掉表前缀**: `SalesPickingOrder`
- **使用业务语义**: 体现业务含义而非技术实现
- **保持简洁**: 易于理解和记忆

#### 示例对比

| Tables (数据层) | Models (业务层) | 说明 |
|----------------|----------------|------|
| `KySalesPickingOrder` | `SalesPickingOrder` | 拣货订单 |
| `KyInventoryBatch` | `InventoryBatch` | 库存批次 |
| `SysWarehouse` | `Warehouse` | 仓库 |
| `KaProduct` | `Product` | 产品 |

---

## 📋 开发规范

### 1. Tables 开发规范

#### ✅ 应该包含的内容

```php
class KySalesPickingOrder extends \yii\db\ActiveRecord
{
    // 1. 状态常量定义
    const STATUS_PENDING = 'pending';
    const STATUS_PICKING = 'picking';
    const STATUS_COMPLETED = 'completed';
    
    // 2. 表名定义
    public static function tableName()
    {
        return 'ky_sales_picking_order';
    }
    
    // 3. 验证规则
    public function rules()
    {
        return [
            [['picking_no', 'order_no'], 'required'],
            // ... 其他规则
        ];
    }
    
    // 4. 字段标签
    public function attributeLabels()
    {
        return [
            'id' => 'ID',
            'picking_no' => 'Picking No',
            // ... 其他标签
        ];
    }
    
    // 5. 基础关联关系
    public function getPicking()
    {
        return $this->hasOne(KySalesPicking::class, ['picking_no' => 'picking_no']);
    }
}
```

#### ❌ 不应该包含的内容

- 复杂的业务计算方法
- 状态管理逻辑
- 自定义行为 (Behaviors)
- 复杂的验证逻辑
- 业务流程方法

### 2. Models 开发规范

#### ✅ 推荐的内容结构

```php
class SalesPickingOrder extends KySalesPickingOrder
{
    use BaseModel;  // 基础功能 trait
    
    // 1. 行为配置
    public function behaviors()
    {
        return [
            'timestamp' => [
                'class' => TimestampBehavior::class,
                'createdAtAttribute' => 'created_at',
                'updatedAtAttribute' => 'updated_at',
            ],
        ];
    }
    
    // 2. 计算属性
    public function getCompletionRate()
    {
        if ($this->total_qty == 0) {
            return 0;
        }
        return round(($this->picked_qty / $this->total_qty) * 100, 2);
    }
    
    // 3. 状态判断方法
    public function isCompleted()
    {
        return $this->picked_qty >= $this->total_qty;
    }
    
    // 4. 业务操作方法
    public function updateProgress($pickedItems, $pickedQty)
    {
        $this->picked_items = $pickedItems;
        $this->picked_qty = $pickedQty;
        
        // 业务逻辑处理...
        
        return $this->save();
    }
    
    // 5. 查询作用域
    public static function findAvailable()
    {
        return static::find()->where(['status' => self::STATUS_PENDING]);
    }
    
    // 6. 关联关系扩展
    public function getOrderItems()
    {
        return $this->hasMany(SalesOrderItem::class, ['order_no' => 'order_no']);
    }
}
```

#### ❌ 避免的内容

- 重复父类已有的基础方法
- 简单的 getter/setter (使用 PHP 魔术方法)
- 过于复杂的业务逻辑 (应移至 Service 层)

### 3. 代码注释规范

#### Tables 注释
```php
/**
 * This is the model class for table "ky_sales_picking_order".
 * 
 * 由 Gii 自动生成，请勿手动修改！
 * 如需添加业务逻辑，请在 common\models\SalesPickingOrder 中实现
 * 
 * @property int $id 主键ID
 * @property string $picking_no 拣货单号
 * // ... 其他属性注释
 */
```

#### Models 注释
```php
/**
 * 拣货订单业务模型
 * 
 * 继承自 KySalesPickingOrder，提供拣货订单相关的业务逻辑
 * 主要功能：
 * - 拣货进度管理
 * - 完成率计算
 * - 状态流转控制
 * 
 * @author 开发者姓名
 * @since 版本号
 */
```

---

## 🎯 最佳实践

### 1. 代码安全实践

#### Gii 重新生成保护
```php
// 在 tables 文件顶部添加注释提醒
/**
 * ⚠️  警告：此文件由 Gii 自动生成！
 * 
 * 重要提醒：
 * 1. 请勿在此文件中添加业务逻辑
 * 2. 表结构变更时会重新生成此文件
 * 3. 所有业务代码请在 common\models 中实现
 * 4. 如需修改，请在对应的 Model 类中重写方法
 */
```

#### 业务代码隔离
```php
// ❌ 错误：在 tables 中添加业务逻辑
class KySalesOrder extends ActiveRecord
{
    public function processOrder()  // 不应该在这里
    {
        // 业务逻辑...
    }
}

// ✅ 正确：在 models 中添加业务逻辑
class SalesOrder extends KySalesOrder
{
    public function processOrder()  // 应该在这里
    {
        // 业务逻辑...
    }
}
```

### 2. 性能优化实践

#### 合理使用继承
```php
class InventoryBatch extends KyInventoryBatch
{
    // 只重写需要的方法，避免不必要的重复
    
    public function rules()
    {
        $rules = parent::rules();
        // 只添加业务特定的验证规则
        $rules[] = ['status', 'validateBusinessStatus'];
        return $rules;
    }
    
    // 自定义验证方法
    public function validateBusinessStatus($attribute, $params)
    {
        // 业务验证逻辑...
    }
}
```

#### 查询优化
```php
class SalesOrder extends KySalesOrder
{
    // 预加载关联关系
    public static function findWithItems($orderNo)
    {
        return static::find()
            ->with(['items', 'customer'])
            ->where(['order_no' => $orderNo])
            ->one();
    }
    
    // 使用查询作用域
    public static function findPendingOrders()
    {
        return static::find()
            ->where(['status' => self::STATUS_PENDING])
            ->orderBy('created_at ASC');
    }
}
```

### 3. 维护性实践

#### 统一基础功能
```php
// BaseModel trait - 统一基础功能
trait BaseModel
{
    /**
     * 获取创建者信息
     */
    public function getCreatedBy()
    {
        return $this->hasOne(User::class, ['id' => 'created_by']);
    }
    
    /**
     * 获取更新者信息
     */
    public function getUpdatedBy()
    {
        return $this->hasOne(User::class, ['id' => 'updated_by']);
    }
    
    /**
     * 格式化时间显示
     */
    public function getFormattedCreatedAt()
    {
        return date('Y-m-d H:i:s', strtotime($this->created_at));
    }
}
```

#### 版本控制友好
```php
// 使用版本控制忽略 tables 目录的变更
// .gitignore 示例
# common/tables/*.php  # 如果需要忽略 Gii 生成的文件

// 或者只提交初始版本，后续变更不提交
# 在团队中统一 Gii 生成规范
```

---

## 💡 代码示例

### 完整示例：库存批次管理

#### 1. Tables 层实现

```php
<?php
namespace common\tables;

use Yii;

/**
 * This is the model class for table "ky_inventory_batch".
 * 
 * ⚠️ 由 Gii 自动生成，请勿手动修改！
 * 业务逻辑请在 common\models\InventoryBatch 中实现
 *
 * @property int $id
 * @property string $kib_number 库存批次号
 * @property string $batch_no 生产批次号
 * @property string $product_sku 产品SKU
 * @property string $sku_barcode SKU条码
 * @property string $warehouse_code 仓库代码
 * @property string $location_code 库位代码
 * @property string $customer_code 客户代码
 * @property int $stock_available 可用库存
 * @property int $stock_freeze 冻结库存
 * @property int $stock_damaged 损坏库存
 * @property string $received_at 入库时间
 * @property string|null $expire_at 过期时间
 * @property string|null $created_at
 * @property string|null $updated_at
 */
class KyInventoryBatch extends \yii\db\ActiveRecord
{
    /**
     * {@inheritdoc}
     */
    public static function tableName()
    {
        return 'ky_inventory_batch';
    }

    /**
     * {@inheritdoc}
     */
    public function rules()
    {
        return [
            [['kib_number', 'batch_no', 'product_sku', 'sku_barcode', 'warehouse_code', 'location_code', 'customer_code'], 'required'],
            [['stock_available', 'stock_freeze', 'stock_damaged'], 'integer'],
            [['received_at', 'expire_at', 'created_at', 'updated_at'], 'safe'],
            [['kib_number'], 'string', 'max' => 32],
            [['batch_no', 'product_sku', 'sku_barcode'], 'string', 'max' => 64],
            [['warehouse_code', 'location_code', 'customer_code'], 'string', 'max' => 20],
            [['stock_available', 'stock_freeze', 'stock_damaged'], 'default', 'value' => 0],
            [['kib_number'], 'unique'],
        ];
    }

    /**
     * {@inheritdoc}
     */
    public function attributeLabels()
    {
        return [
            'id' => 'ID',
            'kib_number' => 'KIB Number',
            'batch_no' => 'Batch No',
            'product_sku' => 'Product Sku',
            'sku_barcode' => 'Sku Barcode',
            'warehouse_code' => 'Warehouse Code',
            'location_code' => 'Location Code',
            'customer_code' => 'Customer Code',
            'stock_available' => 'Stock Available',
            'stock_freeze' => 'Stock Freeze',
            'stock_damaged' => 'Stock Damaged',
            'received_at' => 'Received At',
            'expire_at' => 'Expire At',
            'created_at' => 'Created At',
            'updated_at' => 'Updated At',
        ];
    }
}
```

#### 2. Models 层实现

```php
<?php
namespace common\models;

use common\tables\KyInventoryBatch;
use yii\helpers\ArrayHelper;

/**
 * 库存批次业务模型
 * 
 * 继承自 KyInventoryBatch，提供库存批次管理的业务逻辑
 * 
 * 主要功能：
 * - 库存计算和统计
 * - 过期时间管理
 * - 状态判断
 * - FIFO 查询支持
 * 
 * @author 系统架构师
 * @since 2.0
 */
class InventoryBatch extends KyInventoryBatch
{
    use BaseModel;

    /**
     * 行为配置
     */
    public function behaviors()
    {
        return [
            'timestamp' => [
                'class' => \yii\behaviors\TimestampBehavior::class,
                'createdAtAttribute' => 'created_at',
                'updatedAtAttribute' => 'updated_at',
                'value' => function () {
                    return date('Y-m-d H:i:s');
                },
            ],
        ];
    }

    /**
     * 获取总库存
     * 
     * @return int 总库存数量
     */
    public function getTotalStock()
    {
        return ($this->stock_available ?? 0) + 
               ($this->stock_freeze ?? 0) + 
               ($this->stock_damaged ?? 0);
    }

    /**
     * 获取可操作库存（可用+冻结）
     * 
     * @return int 可操作库存数量
     */
    public function getOperableStock()
    {
        return ($this->stock_available ?? 0) + ($this->stock_freeze ?? 0);
    }

    /**
     * 检查是否即将过期
     * 
     * @param int $days 提前天数，默认30天
     * @return bool
     */
    public function isNearExpiry($days = 30)
    {
        if (!$this->expire_at) {
            return false;
        }
        
        $expireTime = strtotime($this->expire_at);
        $checkTime = time() + ($days * 24 * 3600);
        
        return $expireTime <= $checkTime;
    }

    /**
     * 检查是否已过期
     * 
     * @return bool
     */
    public function isExpired()
    {
        if (!$this->expire_at) {
            return false;
        }
        
        return strtotime($this->expire_at) < time();
    }

    /**
     * 获取批次状态
     * 
     * @return string normal|near_expiry|expired|empty
     */
    public function getStatus()
    {
        if ($this->isExpired()) {
            return 'expired';
        }
        
        if ($this->isNearExpiry()) {
            return 'near_expiry';
        }
        
        if ($this->getTotalStock() <= 0) {
            return 'empty';
        }
        
        return 'normal';
    }

    /**
     * 获取状态中文标签
     * 
     * @return string
     */
    public function getStatusLabel()
    {
        $labels = [
            'normal' => '正常',
            'near_expiry' => '即将过期',
            'expired' => '已过期',
            'empty' => '库存为空',
        ];
        
        return ArrayHelper::getValue($labels, $this->getStatus(), '未知');
    }

    /**
     * 更新库存数量
     * 
     * @param int $availableChange 可用库存变更量
     * @param int $freezeChange 冻结库存变更量
     * @param int $damagedChange 损坏库存变更量
     * @return bool
     */
    public function updateStock($availableChange = 0, $freezeChange = 0, $damagedChange = 0)
    {
        $this->stock_available = max(0, $this->stock_available + $availableChange);
        $this->stock_freeze = max(0, $this->stock_freeze + $freezeChange);
        $this->stock_damaged = max(0, $this->stock_damaged + $damagedChange);
        
        return $this->save();
    }

    /**
     * 获取批次摘要信息
     * 
     * @return array
     */
    public function getSummary()
    {
        return [
            'kib_number' => $this->kib_number,
            'batch_no' => $this->batch_no,
            'product_sku' => $this->product_sku,
            'sku_barcode' => $this->sku_barcode,
            'warehouse_code' => $this->warehouse_code,
            'location_code' => $this->location_code,
            'received_at' => $this->received_at,
            'expire_at' => $this->expire_at,
            'stock_available' => $this->stock_available,
            'stock_freeze' => $this->stock_freeze,
            'stock_damaged' => $this->stock_damaged,
            'total_stock' => $this->getTotalStock(),
            'status' => $this->getStatus(),
            'status_label' => $this->getStatusLabel(),
        ];
    }

    /**
     * 根据产品查找批次（FIFO排序）
     * 
     * @param string $warehouseCode 仓库代码
     * @param string $customerCode 客户代码
     * @param string $productSku 产品SKU
     * @param string $skuBarcode SKU条码
     * @return \yii\db\ActiveQuery
     */
    public static function findByProduct($warehouseCode, $customerCode, $productSku, $skuBarcode)
    {
        return self::find()
            ->where([
                'warehouse_code' => $warehouseCode,
                'customer_code' => $customerCode,
                'product_sku' => $productSku,
                'sku_barcode' => $skuBarcode,
            ])
            ->orderBy('received_at ASC, created_at ASC'); // FIFO 排序
    }

    /**
     * 查找有可用库存的批次（FIFO排序）
     * 
     * @param string $warehouseCode 仓库代码
     * @param string $customerCode 客户代码
     * @param string $productSku 产品SKU
     * @param string $skuBarcode SKU条码
     * @return \yii\db\ActiveQuery
     */
    public static function findAvailableBatches($warehouseCode, $customerCode, $productSku, $skuBarcode)
    {
        return self::findByProduct($warehouseCode, $customerCode, $productSku, $skuBarcode)
            ->andWhere(['>', 'stock_available', 0]);
    }

    /**
     * 查找有冻结库存的批次（FIFO排序）
     * 
     * @param string $warehouseCode 仓库代码
     * @param string $customerCode 客户代码
     * @param string $productSku 产品SKU
     * @param string $skuBarcode SKU条码
     * @return \yii\db\ActiveQuery
     */
    public static function findFreezeBatches($warehouseCode, $customerCode, $productSku, $skuBarcode)
    {
        return self::findByProduct($warehouseCode, $customerCode, $productSku, $skuBarcode)
            ->andWhere(['>', 'stock_freeze', 0]);
    }
}
```

---

## ❓ 常见问题

### Q1: 为什么不直接在 tables 中添加业务方法？

**A**: 主要原因是代码安全性：
- Gii 重新生成会覆盖手动添加的代码
- 数据表结构变更时需要重新生成表类
- 分离关注点，tables 专注数据结构，models 专注业务逻辑

### Q2: 什么时候需要重写父类方法？

**A**: 以下情况需要在 models 中重写：
- 添加业务特定的验证规则
- 扩展关联关系
- 添加计算属性
- 实现业务状态管理

```php
// 示例：扩展验证规则
public function rules()
{
    $rules = parent::rules();
    $rules[] = ['status', 'validateBusinessStatus'];
    return $rules;
}
```

### Q3: 如何处理复杂的业务逻辑？

**A**: 建议使用服务层 (Services)：
- Models: 处理单一实体的业务逻辑
- Services: 处理跨实体、复杂流程的业务逻辑

```php
// 复杂业务逻辑放在 Service 中
class InventoryService
{
    public function processOrderAllocation($orderNo)
    {
        // 跨多个 Model 的复杂逻辑
    }
}
```

### Q4: 如何保证团队开发的一致性？

**A**: 建议制定团队规范：
1. 统一代码审查流程
2. 建立 Gii 生成规范
3. 制定文档更新流程
4. 使用 IDE 模板和代码片段

### Q5: 性能考虑有哪些？

**A**: 主要性能优化点：
- 合理使用查询作用域
- 避免 N+1 查询问题
- 使用 with() 预加载关联
- 缓存计算属性结果

```php
// 优化查询示例
public static function findWithRelations($id)
{
    return static::find()
        ->with(['batchLogs', 'warehouse', 'location'])
        ->where(['id' => $id])
        ->one();
}
```

### Q6: 数据表结构变更时的标准流程是什么？

**A**: 推荐的变更流程：

**第一步：数据库变更**
```sql
-- 例：为订单表添加新字段
ALTER TABLE ky_sales_order ADD COLUMN `priority_level` INT DEFAULT 5;
ALTER TABLE ky_sales_order ADD COLUMN `urgent_flag` TINYINT DEFAULT 0;
```

**第二步：重新生成表类**
```bash
# 使用 Gii 重新生成受影响的表类
# 只需要重新生成 common/tables 中的类
# common/models 中的业务代码不受影响
```

**第三步：扩展业务逻辑（可选）**
```php
// 在 common/models 中添加新功能
class SalesOrder extends KySalesOrder 
{
    // 新增常量
    const PRIORITY_URGENT = 1;
    const PRIORITY_NORMAL = 5;
    
    // 新增业务方法
    public function isUrgent() 
    {
        return $this->urgent_flag == 1 || $this->priority_level <= 2;
    }
}
```

### Q7: 如何处理 Gii 生成的代码不符合要求的情况？

**A**: 几种处理方式：

1. **自定义 Gii 模板**
```php
// 创建自定义模板文件
// 在 Gii 配置中使用自定义模板
'generators' => [
    'model' => [
        'class' => 'yii\gii\generators\model\Generator',
        'templates' => [
            'custom' => '@app/templates/gii/model',
        ]
    ]
]
```

2. **表类后处理**
```php
// 如果确实需要在表类中添加内容
// 建议使用 traits 方式
trait CustomTableBehavior 
{
    public function getCustomAttribute() 
    {
        // 自定义逻辑
    }
}

class KySalesOrder extends ActiveRecord 
{
    use CustomTableBehavior;
    // Gii 生成的内容...
}
```

### Q8: 什么情况下可以直接修改 tables 中的代码？

**A**: 只有以下极少数情况：

1. **修改基础关联关系**（当 Gii 生成有误时）
2. **添加数据库层面的约束验证**
3. **修复 Gii 生成的明显错误**

但即使在这些情况下，也要：
- 做好代码备份
- 在注释中标明修改原因
- 考虑是否可以通过其他方式解决

### Q9: 如何处理表名前缀不一致的问题？

**A**: 统一命名策略：

```php
// 建议的命名映射规则
class NamingHelper 
{
    protected static $tableToModelMap = [
        'KySalesOrder' => 'SalesOrder',           // 去掉 Ky 前缀
        'SysWarehouse' => 'Warehouse',            // 去掉 Sys 前缀
        'KaProduct' => 'Product',                 // 去掉 Ka 前缀
        'OmsMenu' => 'Menu',                      // 去掉 Oms 前缀
    ];
    
    public static function getModelName($tableName) 
    {
        return self::$tableToModelMap[$tableName] ?? $tableName;
    }
}
```

### Q10: 如何在业务模型中添加事件处理？

**A**: 推荐在 models 中处理业务事件：

```php
class SalesOrder extends KySalesOrder 
{
    public function init()
    {
        parent::init();
        
        // 业务事件绑定
        $this->on(self::EVENT_BEFORE_INSERT, [$this, 'beforeOrderCreate']);
        $this->on(self::EVENT_AFTER_UPDATE, [$this, 'afterOrderUpdate']);
    }
    
    public function beforeOrderCreate($event)
    {
        // 订单创建前的业务处理
        $this->generateOrderNumber();
    }
    
    public function afterOrderUpdate($event) 
    {
        // 订单更新后的业务处理
        if ($this->isAttributeChanged('order_status')) {
            $this->notifyStatusChange();
        }
    }
}
```

### Q11: 如何处理大量计算属性的性能问题？

**A**: 性能优化策略：

```php
class InventoryBatch extends KyInventoryBatch 
{
    private $_totalStock = null;  // 缓存计算结果
    private $_summaryData = null;
    
    /**
     * 带缓存的计算属性
     */
    public function getTotalStock()
    {
        if ($this->_totalStock === null) {
            $this->_totalStock = ($this->stock_available ?? 0) + 
                                ($this->stock_freeze ?? 0) + 
                                ($this->stock_damaged ?? 0);
        }
        return $this->_totalStock;
    }
    
    /**
     * 重置缓存
     */
    public function afterSave($insert, $changedAttributes)
    {
        parent::afterSave($insert, $changedAttributes);
        
        // 库存相关字段变更时重置缓存
        if (array_intersect(['stock_available', 'stock_freeze', 'stock_damaged'], array_keys($changedAttributes))) {
            $this->_totalStock = null;
            $this->_summaryData = null;
        }
    }
    
    /**
     * 批量计算优化
     */
    public static function batchCalculateStock($batches)
    {
        $results = [];
        foreach ($batches as $batch) {
            $results[] = [
                'kib_number' => $batch->kib_number,
                'total_stock' => $batch->getTotalStock(),
                'status' => $batch->getStatus(),
            ];
        }
        return $results;
    }
}
```

### Q12: 如何确保业务模型的单元测试覆盖？

**A**: 测试策略建议：

```php
// tests/unit/models/SalesOrderTest.php
class SalesOrderTest extends \Codeception\Test\Unit
{
    public function testStatusTransition()
    {
        $order = new SalesOrder();
        $order->order_status = SalesOrder::STATUS_PEND;
        
        // 测试业务逻辑
        $this->assertTrue($order->canConfirm());
        $this->assertFalse($order->canShip());
        
        // 测试状态转换
        $order->confirmOrder();
        $this->assertEquals(SalesOrder::STATUS_CONF, $order->order_status);
    }
    
    public function testCalculatedProperties()
    {
        $order = new SalesOrder();
        $order->order_status = SalesOrder::STATUS_SHIP;
        
        // 测试计算属性
        $this->assertEquals('已发货', $order->getStatusLabel());
        $this->assertEquals(90, $order->getProgressPercentage());
    }
    
    public function testQueryScopes()
    {
        // 测试查询作用域
        $query = SalesOrder::findPending();
        $this->assertInstanceOf(\yii\db\ActiveQuery::class, $query);
        
        $expectedCondition = ['order_status' => SalesOrder::STATUS_PEND];
        $this->assertEquals($expectedCondition, $query->where);
    }
}
```

### Q13: 如何处理业务模型之间的循环依赖？

**A**: 避免循环依赖的方法：

```php
// ❌ 避免这样的循环依赖
class SalesOrder extends KySalesOrder 
{
    public function getInventoryStatus()
    {
        return InventoryBatch::getOrderAllocationStatus($this->order_no);  // 不好
    }
}

class InventoryBatch extends KyInventoryBatch 
{
    public function getRelatedOrders()
    {
        return SalesOrder::findByInventory($this->kib_number);  // 循环依赖
    }
}

// ✅ 推荐的解决方案
// 1. 使用服务层处理跨模型逻辑
class OrderInventoryService 
{
    public static function getOrderAllocationStatus($orderNo)
    {
        // 在服务层处理跨模型的复杂逻辑
    }
    
    public static function getInventoryOrderRelations($kibNumber)
    {
        // 在服务层处理关联查询
    }
}

// 2. 使用事件机制解耦
class SalesOrder extends KySalesOrder 
{
    public function afterSave($insert, $changedAttributes)
    {
        parent::afterSave($insert, $changedAttributes);
        
        // 通过事件通知，而不是直接调用
        $this->trigger('orderStatusChanged', new OrderStatusChangedEvent([
            'order' => $this,
            'oldStatus' => $changedAttributes['order_status'] ?? null
        ]));
    }
}
```

### Q14: 大型项目中如何组织业务模型的目录结构？

**A**: 推荐的目录组织方式：

```
common/models/
├── BaseModel.php              # 基础模型 trait
├── BaseForm.php               # 基础表单类
├── order/                     # 订单相关模型
│   ├── SalesOrder.php
│   ├── SalesOrderItem.php
│   ├── SalesOrderAddress.php
│   └── SalesPickingOrder.php
├── inventory/                 # 库存相关模型
│   ├── InventoryBatch.php
│   ├── InventoryLocation.php
│   ├── InventoryTransaction.php
│   └── InventoryWarehouse.php
├── picking/                   # 拣货相关模型
│   ├── SalesPicking.php
│   ├── SalesPickingDetail.php
│   └── SalesPickingOrder.php
├── system/                    # 系统配置模型
│   ├── User.php
│   ├── Warehouse.php
│   ├── Customer.php
│   └── Dict.php
└── forms/                     # 表单模型
    ├── OrderCreateForm.php
    ├── InventorySearchForm.php
    └── PickingAssignForm.php
```

并在各个模型中使用命名空间：

```php
<?php
namespace common\models\order;

use common\tables\KySalesOrder;
use common\models\BaseModel;

class SalesOrder extends KySalesOrder
{
---

## 📝 总结

这种分层架构设计确保了：

1. **🔒 代码安全**: Gii 重新生成不影响业务代码
2. **🧩 职责清晰**: 数据层与业务层分离
3. **🚀 易于维护**: 代码结构清晰，便于扩展
4. **⚡ 开发效率**: 减少重复工作，提高协作效率

通过遵循这些架构原则和开发规范，我们可以构建出高质量、可维护的企业级应用系统。 