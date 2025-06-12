# 3PL-WMS Lagerverwaltungssystem

## 🌐 Sprachversionen / Language Versions
- [中文 (Chinese)](README.md)
- [English](README_EN.md)
- [Deutsch (German)](README_DE.md) - Aktuelle Seite
- [日本語 (Japanese)](README_JA.md)

---

## 📋 Projektübersicht

Dieses Projekt ist ein Kernmodul eines Lagerverwaltungssystems auf Unternehmensebene, das sich auf Auftragsbearbeitung, Bestandsverwaltung, Kommissionieroperationen und Versandprozesse konzentriert. Das System verwendet modernes architektonisches Design zur Unterstützung von Lagerbetriebsszenarien mit hoher Parallelität und hoher Verfügbarkeit und folgt strikt dem FIFO (First In, First Out)-Prinzip für die Bestandsverwaltung.

## 🔄 Vollständiger Geschäftsprozess

### Phase 1: Auftragsbearbeitung und Kommissionierdetails-Generierung

```
1. Auftragsübermittlung (STATUS_PEND)
   ↓
2. Auftragsgenehmigung (STATUS_CONF)
   ↓
3. Vor-Kommissionierung Sortierung + FIFO Bestandseinfrierung (STATUS_PICK)
   ↓
4. Kommissionierdetails generieren (picking_no = null, status = pending)
```

**Hauptoperationen:**
- Auftragsinformationsvalidierung
- FIFO Bestandsvorprüfung
- Statusflusskontrolle

### Phase 2: Wellen-Kommissionierung und Ausführung

```
5. Kommissionierregeln Batch-Bindung → Kommissionierauftragsnummer ergänzen → Kommissionierer kommissioniert → Verpackung → Versand
6. Unterstützung mehrerer Wellenstrategien (nach Zeit, Lagerplatz, Kunde, Priorität, etc.)
7. Intelligente Zusammenführung gleicher SKUs, Optimierung der Kommissionierwege
8. Echtzeitverfolgung des Kommissionierfortschritts
```

**Kernfunktionen:**
- **FIFO Bestandseinfrierung** - Einfrieren von Batch-Beständen basierend auf dem First-In-First-Out-Prinzip nach Eingangszeitpunkt
- **Intelligente Wegoptimierung** - Optimierung der Kommissionierwege durch Sortierung der Lagerplatzcodes
- **Mehrdimensionale Bestandsverwaltung** - Dreistufige Bestandsstatistiken: Batch, Lagerplatz, Lager
- **Wellen-Kommissionierung** - Zuweisung von Kommissionierauftragsnummern basierend auf Wellenregeln

### Phase 3: Versandbearbeitung

```
9. Automatische Versandbearbeitung (STATUS_SHIP)
    ↓
10. FIFO Reduzierung eingefrorener Bestände
    ↓
11. Logistikübergabe
    ↓
12. Kundenzustellung (STATUS_FINISH)
```

## 🛠️ Technische Funktionen

### FIFO Bestandsverwaltung (Kernfunktion)

#### 1. FIFO-Prinzip-Implementierung

```php
// Alle Bestandsoperationen werden nach Eingangszeitpunkt sortiert, um First-In-First-Out zu gewährleisten
->orderBy('received_at ASC, created_at ASC')
```

#### 2. FIFO Bestandseinfrierung

- **Auto-Zuweisungseinfrierung** - `autoAllocateStockFreeze()` wählt automatisch Batches zur Einfrierung basierend auf dem FIFO-Prinzip aus
- **Spezifische Lagerplatz-Einfrierung** - `freezeLocationStockByFifo()` friert innerhalb spezifischer Lagerplätze basierend auf dem FIFO-Prinzip ein
- **Batch-Rückverfolgbarkeit** - Vollständige Aufzeichnung des Einfrierungsstatus und der Eingangszeitpunkte jedes Batches

```php
// Beispiel: FIFO Bestandseinfrierung
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
    "Auftrags-Bestandseinfrierung"
);
```

#### 3. FIFO Bestandsfreigabe

- **Auto-Freigabe** - `autoReleaseStockFreeze()` gibt eingefrorene Bestände basierend auf dem FIFO-Prinzip frei
- **Batch-Abgleich** - Priorisierung der Freigabe der frühesten eingefrorenen Batches
- **Bestandsausgleich** - Automatische Aktualisierung dreistufiger Bestandsstatistiken: Batch, Lagerplatz, Lager

#### 4. FIFO Bestandsabzug

- **Auto-Abzug** - `autoDeductStockFromLocation()` zieht verfügbare Bestände basierend auf dem FIFO-Prinzip ab
- **Versandabzug** - Priorisierung des Abzugs der frühesten eingegangenen Batches beim Versand
- **Kostenrechnung** - Unterstützung der FIFO-Kostenrechnungsmethode

#### 5. FIFO-Vorteile

- **Bestandsumschlagsoptimierung** - Gewährleistung, dass früher eingegangene Waren zuerst verwendet werden, Vermeidung von Bestandsakkumulation
- **Haltbarkeitsverwaltung** - Für Waren mit Haltbarkeitsdatum kann FIFO die Ablaufrisiken reduzieren
- **Genaue Kostenrechnung** - Verwendung des Bestands in Eingangsreihenfolge für genauere Kostenrechnung
- **Bestandsrückverfolgbarkeit** - Vollständige Batch-Rückverfolgungskette für Qualitätsproblemuntersuchungen
- **Compliance-Anforderungen** - Erfüllung der FIFO-Compliance-Anforderungen in bestimmten Branchen

#### 6. Anwendungsszenarien

- **Lebensmittel & Getränke** - Strenge Haltbarkeitsverwaltung
- **Pharmaindustrie** - Arzneimittel-Batch-Verwaltung und Ablaufdatumskontrolle
- **Chemische Produkte** - Chemische Stabilität und Sicherheitsverwaltung
- **Elektronische Produkte** - Vermeidung von Komponentenalterung und technischer Obsoleszenz
- **Textilindustrie** - Bestandsumschlag für saisonale Waren

### Redis-Optimierung

- **Verteilte Sperren** - Verhinderung von parallelen Operationskonflikten
- **Caching-Mechanismus** - Verbesserung der Abfrageleistung
- **Event-Publishing** - Echtzeitstatus-Benachrichtigungen
- **Fortschrittsverfolgung** - Echtzeit-Kommissionierfortschritt

#### Parallelitätskontrollmechanismus

Das System verwendet Redis-verteilte Sperren, um Datenkonsistenz in Hochparallelitätsszenarien zu gewährleisten:

```php
// Batch-Bestandssperre - feinste Granularität
$lockKey = "batch_stock_lock:{$warehouseCode}:{$customerCode}:{$productSku}:{$skuBarcode}:{$lotNumber}";

// Lagerplatz-Bestandssperre - mittlere Granularität
$lockKey = "location_stock_lock:{$warehouseCode}:{$locationCode}:{$customerCode}:{$productSku}:{$skuBarcode}";

// Lager-Bestandssperre - grobe Granularität
$lockKey = "warehouse_stock_lock:{$warehouseCode}:{$customerCode}:{$productSku}:{$skuBarcode}";
```

**Sperr-Eigenschaften:**
- **Sperr-Timeout**: 30 Sekunden, Verhinderung von Deadlocks
- **Max. Wartezeit**: 10 Sekunden, Vermeidung langer Blockierungen
- **Atomare Operationen**: Gewährleistung der Atomarität von Bestandsaktualisierungen
- **Negativbestandsprüfung**: Vermeidung von Überverkäufen
- **Auto-Wiederholung**: Automatische Wiederholung bei temporären Fehlern

**Parallelitätssicherheitsgarantien:**
- Mehrere Aufträge frieren gleichzeitig Bestände ein, sequenzielle Ausführung
- Datenkonsistenz bei parallelen Ein- und Ausgangsoperationen
- Verhinderung von Data-Race bei Batch-Bestandsaktualisierungen
- Echtzeitgenaue Bestandsstatistiken, kein Datenverlust

### Intelligente Wegoptimierung

```php
// Sortierung nach Lagerplatzcode zur Optimierung der Kommissionierwege
usort($details, function($a, $b) {
    return strcmp($a['location_code'], $b['location_code']);
});
```

## 📊 Verwendungsbeispiele

### 1. FIFO Bestandseinfrierungs-Beispiel

```php
$inventoryService = new InventoryService();

// Aktuellen Bestandsstatus anzeigen (nach FIFO sortiert)
$stockStatus = $inventoryService->getDetailedStockStatus('WH01', 'CUST001', 'SKU001', 'BAR001');

// FIFO Bestandseinfrierung ausführen
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
    "FIFO Bestandseinfrierungs-Demonstration"
);

// System wählt automatisch Batches zur Einfrierung basierend auf First-In-First-Out-Prinzip nach Eingangszeitpunkt aus
```

### 2. Auftragsbearbeitungsphase

```php
$orderService = new OrderService();
$result = $orderService->performPrePickSorting('SO202412150001', true);

// Rückgabeergebnisse umfassen:
// - Kommissionierdetails generieren (FIFO-Prinzip befolgen)
```

### 3. Wellen-Kommissionierungsphase

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

// Rückgabeergebnisse umfassen:
// - Kommissionierauftrag generieren
```

### 4. Manuelle Detail-Zuweisung

```php
$waveService = new WavePickingService();
$assignResult = $waveService->assignDetailsToPickingManually(
    [1, 2, 3], // Detail-ID-Array
    'WWH0120241215000001' // Kommissionierauftragsnummer
);

// Rückgabeergebnisse umfassen:
// - Anzahl erfolgreich zugewiesener Details
```

### 5. Kommissionierausführungsphase

```php
$pickingService = new PickingService();
$pickingService->startPicking('WWH0120241215000001', 'picker001');
$pickingService->scanAndPick('WWH0120241215000001', 'SKU001', 10, 'BATCH001');
$pickingService->completePicking('WWH0120241215000001');
```

## 🔧 Konfigurationsoptionen

### Teilversand-Konfiguration

```php
$config = [
    'enabled' => true,                    // Teilversand aktivieren
    'min_completion_rate' => 0.8,         // Mindestabschlussrate 80%
    'max_partial_times' => 3,             // Maximale Teilversandzeiten
    'auto_ship_threshold' => 0.9,         // Auto-Versandschwelle 90%
    'require_approval' => false,          // Keine Genehmigung erforderlich
    'notification_enabled' => true,       // Benachrichtigungen aktivieren
];

$orderService->setPartialShipmentConfig($config, 'CUSTOMER001', 'WH001');
```

### Kommissioniertyp-Konfiguration

```php
$pickingOptions = [
    'picking_type' => PickingService::TYPE_NORMAL,  // Normale Kommissionierung
    'picker' => 'picker001',                        // Zugewiesener Kommissionierer
    'priority' => 1                                 // Priorität
];
```

## 📈 Leistungsoptimierung

### 1. Redis-Caching-Strategie

- **Auftragsinformations-Cache** - 5 Minuten TTL
- **Kommissionierfortschritts-Cache** - 30 Minuten TTL
- **Konfigurationsinformations-Cache** - 1 Stunde TTL
- **FIFO-Batch-Cache** - 10 Minuten TTL

### 2. Datenbankoptimierung

- **Index-Optimierung** - Composite-Indizes für Schlüsselfelder erstellen
- **FIFO-Index** - `(warehouse_code, customer_code, product_sku, received_at, created_at)`
- **Batch-Operationen** - Reduzierung der Datenbankroundtrips
- **Paginierte Abfragen** - Paginierte Verarbeitung für große Datenmengen

### 3. Parallelitätskontrolle

- **Verteilte Sperren** - Verhinderung paralleler Konflikte
- **Atomare Operationen** - Gewährleistung der Datenkonsistenz
- **Wiederholungsmechanismus** - Behandlung temporärer Fehler

## 🚀 Bereitstellungsanweisungen

### Umgebungsanforderungen

- PHP 7.4+
- Yii2 Framework
- MySQL 8.0+
- Redis 6.0+

## 🎯 Zukunfts-Roadmap

1. **KI-Wegoptimierung** - Maschinelles Lernen zur Optimierung der Kommissionierwege
2. **Automatisierungsintegration** - Integration mit automatisierten Geräten
3. **Mobile Unterstützung** - Entwicklung mobiler Kommissionierungsanwendungen
4. **Datenanalyse** - Kommissioniereffizienz-Analyseberichte
5. **Multi-Lager-Unterstützung** - Lagerübergreifende Transferfunktionalität

## 📞 Technischer Support

Bei Fragen oder Vorschlägen wenden Sie sich bitte an das Entwicklungsteam.

<img src="./docs/image/wechat.png" alt="donate" width="200" />

--- 