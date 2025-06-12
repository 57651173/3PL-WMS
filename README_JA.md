# 3PL-WMS 倉庫管理システム

## 🌐 言語バージョン / Language Versions
- [中文 (Chinese)](README.md)
- [English](README_EN.md)
- [Deutsch (German)](README_DE.md)
- [日本語 (Japanese)](README_JA.md) - 現在のページ

---

## 📋 プロジェクト概要

本プロジェクトは、注文処理、在庫管理、ピッキング作業、出荷プロセスに焦点を当てた企業レベルの倉庫管理システムのコアモジュールです。システムは高並行性、高可用性の倉庫運営シナリオをサポートする現代的なアーキテクチャ設計を採用し、在庫管理においてFIFO（先入先出）原則を厳格に遵守しています。

## 🔄 完全なビジネスプロセス

### フェーズ1：注文処理とピッキング詳細生成

```
1. 注文提出 (STATUS_PEND)
   ↓
2. 注文承認 (STATUS_CONF)
   ↓
3. プレピッキングソート + FIFO在庫凍結 (STATUS_PICK)
   ↓
4. ピッキング詳細生成 (picking_no = null, status = pending)
```

**主要操作：**
- 注文情報検証
- FIFO在庫事前チェック
- ステータスフロー制御

### フェーズ2：ウェーブピッキングと実行

```
5. ピッキングルールバッチバインディング → ピッキング注文番号補完 → ピッカーピッキング → 梱包 → 出荷
6. 複数のウェーブ戦略をサポート（時間、ロケーション、顧客、優先度など）
7. 同一SKUのインテリジェント統合、ピッキングパスの最適化
8. リアルタイムピッキング進捗追跡
```

**コア機能：**
- **FIFO在庫凍結** - 入庫時間による先入先出原則に基づくバッチ在庫の凍結
- **インテリジェントパス最適化** - ロケーションコードのソートによるピッキングパスの最適化
- **多次元在庫管理** - バッチ、ロケーション、倉庫の3レベル在庫統計
- **ウェーブピッキング** - ウェーブルールに基づくピッキング注文番号の割り当て

### フェーズ3：出荷処理

```
9. 自動出荷処理 (STATUS_SHIP)
    ↓
10. FIFO凍結在庫削減
    ↓
11. 物流引き渡し
    ↓
12. 顧客配送 (STATUS_FINISH)
```

## 🛠️ 技術的特徴

### FIFO在庫管理（コア機能）

#### 1. FIFO原則実装

```php
// すべての在庫操作は入庫時間でソートされ、先入先出を保証
->orderBy('received_at ASC, created_at ASC')
```

#### 2. FIFO在庫凍結

- **自動割り当て凍結** - `autoAllocateStockFreeze()` FIFO原則に基づいて凍結用バッチを自動選択
- **指定ロケーション凍結** - `freezeLocationStockByFifo()` 指定ロケーション内でFIFO原則に基づいて凍結
- **バッチトレーサビリティ** - 各バッチの凍結状況と入庫時間の完全記録

```php
// 例：FIFO在庫凍結
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
    "注文在庫凍結"
);
```

#### 3. FIFO在庫解放

- **自動解放** - `autoReleaseStockFreeze()` FIFO原則に基づいて凍結在庫を解放
- **バッチマッチング** - 最も早く凍結されたバッチの解放を優先
- **在庫バランス** - バッチ、ロケーション、倉庫の3レベル在庫統計を自動更新

#### 4. FIFO在庫控除

- **自動控除** - `autoDeductStockFromLocation()` FIFO原則に基づいて利用可能在庫を控除
- **出荷控除** - 出荷時に最も早く入庫したバッチを優先的に控除
- **原価計算** - FIFO原価計算方法をサポート

#### 5. FIFOの利点

- **在庫回転最適化** - 早期入庫商品の優先使用を保証し、在庫蓄積を回避
- **賞味期限管理** - 賞味期限のある商品について、FIFOは期限切れリスクを削減
- **正確な原価計算** - 入庫順序での在庫使用により、より正確な原価計算
- **在庫トレーサビリティ** - 品質問題調査のための完全なバッチトレーサビリティチェーン
- **コンプライアンス要件** - 特定業界でのFIFOコンプライアンス要件を満たす

#### 6. 適用シナリオ

- **食品・飲料** - 厳格な賞味期限管理
- **製薬業界** - 薬品バッチ管理と有効期限制御
- **化学製品** - 化学安定性と安全性管理
- **電子製品** - 部品の老化と技術的陳腐化の回避
- **繊維業界** - 季節商品の在庫回転

### Redis最適化

- **分散ロック** - 並行操作競合の防止
- **キャッシュメカニズム** - クエリパフォーマンスの向上
- **イベント発行** - リアルタイムステータス通知
- **進捗追跡** - リアルタイムピッキング進捗

#### 並行制御メカニズム

システムは高並行シナリオでのデータ整合性を保証するためにRedis分散ロックを使用：

```php
// バッチ在庫ロック - 最細粒度
$lockKey = "batch_stock_lock:{$warehouseCode}:{$customerCode}:{$productSku}:{$skuBarcode}:{$lotNumber}";

// ロケーション在庫ロック - 中粒度
$lockKey = "location_stock_lock:{$warehouseCode}:{$locationCode}:{$customerCode}:{$productSku}:{$skuBarcode}";

// 倉庫在庫ロック - 粗粒度
$lockKey = "warehouse_stock_lock:{$warehouseCode}:{$customerCode}:{$productSku}:{$skuBarcode}";
```

**ロック特性：**
- **ロックタイムアウト**：30秒、デッドロック防止
- **最大待機時間**：10秒、長時間ブロッキング回避
- **アトミック操作**：在庫更新のアトミック性を保証
- **負在庫チェック**：過剰販売防止
- **自動リトライ**：一時的エラーの自動リトライ

**並行安全保証：**
- 複数注文が同時に在庫を凍結する際の順次実行
- 入庫・出庫操作が並行する際のデータ整合性維持
- バッチ在庫更新時のデータ競合防止
- リアルタイム正確な在庫統計、データ損失なし

### インテリジェントパス最適化

```php
// ロケーションコードでソートしてピッキングパスを最適化
usort($details, function($a, $b) {
    return strcmp($a['location_code'], $b['location_code']);
});
```

## 📊 使用例

### 1. FIFO在庫凍結例

```php
$inventoryService = new InventoryService();

// 現在の在庫状況を表示（FIFOソート）
$stockStatus = $inventoryService->getDetailedStockStatus('WH01', 'CUST001', 'SKU001', 'BAR001');

// FIFO在庫凍結実行
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
    "FIFO在庫凍結デモンストレーション"
);

// システムは入庫時間による先入先出原則に基づいて凍結用バッチを自動選択
```

### 2. 注文処理フェーズ

```php
$orderService = new OrderService();
$result = $orderService->performPrePickSorting('SO202412150001', true);

// 戻り値に含まれるもの：
// - ピッキング詳細生成（FIFO原則に従う）
```

### 3. ウェーブピッキングフェーズ

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

// 戻り値に含まれるもの：
// - ピッキング注文生成
```

### 4. 手動詳細割り当て

```php
$waveService = new WavePickingService();
$assignResult = $waveService->assignDetailsToPickingManually(
    [1, 2, 3], // 詳細ID配列
    'WWH0120241215000001' // ピッキング注文番号
);

// 戻り値に含まれるもの：
// - 正常に割り当てられた詳細数
```

### 5. ピッキング実行フェーズ

```php
$pickingService = new PickingService();
$pickingService->startPicking('WWH0120241215000001', 'picker001');
$pickingService->scanAndPick('WWH0120241215000001', 'SKU001', 10, 'BATCH001');
$pickingService->completePicking('WWH0120241215000001');
```

## 🔧 設定オプション

### 部分出荷設定

```php
$config = [
    'enabled' => true,                    // 部分出荷を有効化
    'min_completion_rate' => 0.8,         // 最小完了率 80%
    'max_partial_times' => 3,             // 最大部分出荷回数
    'auto_ship_threshold' => 0.9,         // 自動出荷閾値 90%
    'require_approval' => false,          // 承認不要
    'notification_enabled' => true,       // 通知を有効化
];

$orderService->setPartialShipmentConfig($config, 'CUSTOMER001', 'WH001');
```

### ピッキングタイプ設定

```php
$pickingOptions = [
    'picking_type' => PickingService::TYPE_NORMAL,  // 通常ピッキング
    'picker' => 'picker001',                        // 割り当てピッカー
    'priority' => 1                                 // 優先度
];
```

## 📈 パフォーマンス最適化

### 1. Redisキャッシュ戦略

- **注文情報キャッシュ** - 5分TTL
- **ピッキング進捗キャッシュ** - 30分TTL
- **設定情報キャッシュ** - 1時間TTL
- **FIFOバッチキャッシュ** - 10分TTL

### 2. データベース最適化

- **インデックス最適化** - キーフィールドの複合インデックス構築
- **FIFOインデックス** - `(warehouse_code, customer_code, product_sku, received_at, created_at)`
- **バッチ操作** - データベースラウンドトリップの削減
- **ページネーションクエリ** - 大容量データのページネーション処理

### 3. 並行制御

- **分散ロック** - 並行競合の防止
- **アトミック操作** - データ整合性の保証
- **リトライメカニズム** - 一時的エラーの処理

## 🚀 デプロイメント手順

### 環境要件

- PHP 7.4+
- Yii2 Framework
- MySQL 8.0+
- Redis 6.0+

## 🎯 将来のロードマップ

1. **AIパス最適化** - 機械学習によるピッキングパス最適化
2. **自動化統合** - 自動化設備との統合
3. **モバイルサポート** - モバイルピッキングアプリケーションの開発
4. **データ分析** - ピッキング効率分析レポート
5. **マルチ倉庫サポート** - 倉庫間転送機能

## 📞 技術サポート

ご質問やご提案がございましたら、開発チームまでお問い合わせください。

<img src="./docs/image/wechat.png" alt="donate" width="200" />

--- 