# 商品数量调整与库存流水一致性分析

## 1. 核心概念与数据结构

### 1.1 商品资料 (items 表)

**模型文件**: [Item.php](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Models/Item.php)

#### 1.1.1 核心字段说明

| 字段 | 类型 | 说明 | 常量定义 |
|------|------|------|----------|
| `stock_type` | int | **库存管控开关**：决定该商品是否参与所有库存变动操作 | `HAS_STOCK` = 0 (有库存), `HAS_NO_STOCK` = 1 (无库存) |
| `item_type` | int | **商品业务类型**：决定商品的业务行为 | `ITEM` = 0, `ITEM_KIT` = 1, `ITEM_AMOUNT_ENTRY` = 2, `ITEM_TEMP` = 3 |
| `receiving_quantity` | decimal | **默认收货换算倍率**：收货时每单位采购对应的实际入库数量 | - |
| `reorder_level` | decimal | 库存预警水平 | - |
| `qty_per_pack` | decimal | 每包数量（销售包装） | - |

#### 1.1.2 关键常量定义

**位置**: [Constants.php](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Config/Constants.php#L107-L114)

```php
const HAS_STOCK = 0;      // 参与库存管理
const HAS_NO_STOCK = 1;   // 不参与库存管理（服务、劳务等）
const ITEM = 0;           // 普通商品
const ITEM_KIT = 1;       // 商品套装（组合品）
const ITEM_AMOUNT_ENTRY = 2;  // 金额输入商品（无固定数量）
const ITEM_TEMP = 3;      // 临时商品（一次性格销售商品）
```

#### 1.1.3 stock_type 与 item_type 的真实约束关系

**核心代码**: [Items.php::postSave()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Controllers/Items.php#L661-L665)

```php
if ($item_data['item_type'] == ITEM_TEMP) {
    $item_data['stock_type'] = HAS_NO_STOCK;      // 强制无库存
    $item_data['receiving_quantity'] = 0;          // 强制收货倍率为0
    $item_data['reorder_level'] = 0;               // 强制无预警
}
```

**强制约束关系表**：

| item_type | stock_type 可选项 | receiving_quantity 约束 | 说明 |
|-----------|-----------------|------------------------|------|
| `ITEM` (0) | `HAS_STOCK` 或 `HAS_NO_STOCK` | 用户输入，若为0则强制设为1 | 普通商品，可灵活配置 |
| `ITEM_KIT` (1) | `HAS_STOCK` 或 `HAS_NO_STOCK` | 用户输入，若为0则强制设为1 | 套装本身一般 HAS_NO_STOCK，通过子商品扣库存 |
| `ITEM_AMOUNT_ENTRY` (2) | `HAS_STOCK` 或 `HAS_NO_STOCK` | 用户输入，若为0则强制设为1 | 金额商品一般 HAS_NO_STOCK |
| `ITEM_TEMP` (3) | **强制 HAS_NO_STOCK**（代码覆盖） | **强制设为 0** | 临时商品永远不参与库存管理 |

**receiving_quantity 的自动修正**:
**代码**: [Items.php::postSave()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Controllers/Items.php#L628-L630)

```php
if ($receiving_quantity === 0.0 && $item_type !== ITEM_TEMP) {
    $receiving_quantity = 1;   // 只要不是临时商品，就不允许 receiving_quantity=0
}
```

### 1.2 库存数量 (item_quantities 表)

**模型文件**: [Item_quantity.php](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Models/Item_quantity.php)

| 字段 | 类型 | 说明 |
|------|------|------|
| `item_id` | int | 商品ID (联合主键) |
| `location_id` | int | 仓库位置ID (联合主键) |
| `quantity` | decimal | 当前库存数量 |

**核心方法**:
- `save_value()`: 直接设置绝对数量（用于新建、手动调整、CSV导入）
- `change_quantity()`: 增量更新（用于删除回滚操作）

### 1.3 库存流水 (inventory 表)

**模型文件**: [Inventory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Models/Inventory.php)

| 字段 | 类型 | 说明 |
|------|------|------|
| `trans_id` | int | 流水ID (自增主键) |
| `trans_items` | int | 商品ID |
| `trans_user` | int | 操作人ID |
| `trans_date` | datetime | 操作时间 |
| `trans_comment` | varchar | 备注说明 |
| `trans_inventory` | decimal | 库存变化量 (正=入库, 负=出库) |
| `trans_location` | int | 仓库位置ID |

---

## 2. receiving_quantity 对库存流水的影响机制

### 2.1 receiving_quantity 的本质

`receiving_quantity` 是 **「采购包装单位 → 库存基本单位」的换算倍率**，仅在收货场景生效。

**例**: 商品A采购时按「箱」进货，每箱有24瓶。设置 `receiving_quantity = 24`。收货时录入采购数量=5箱，则实际入库 5×24=120 瓶。

### 2.2 收货时的计算流程

**代码位置**: [Receiving.php::save_value()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Models/Receiving.php#L154-L183)

```php
// 步骤1：计算实际入库数量
$items_received = $item_data['receiving_quantity'] != 0
    ? $item_data['quantity'] * $item_data['receiving_quantity']    // 使用换算倍率
    : $item_data['quantity'];                                      // 倍率为0时直接使用数量

// 步骤2：更新 item_quantities（加 items_received）
$item_quantity->save_value([
    'quantity' => $item_quantity_value->quantity + $items_received,
    // ...
]);

// 步骤3：插入 inventory 流水（trans_inventory = items_received）
$inv_data = [
    'trans_comment'   => 'RECV ' . $receiving_id,
    'trans_inventory' => $items_received    // 注意：不是 quantity，是 items_received
];
$inventory->insert($inv_data, false);
```

### 2.3 删除收货时的回滚计算

**代码位置**: [Receiving.php::delete_value()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Models/Receiving.php#L238-L244)

```php
// 回滚时同样使用换算倍率
$inv_data = [
    'trans_comment'   => 'Deleting receiving ' . $receiving_id,
    'trans_inventory' => $item['quantity_purchased'] * (-$item['receiving_quantity'])
];
$inventory->insert($inv_data, false);

$item_quantity->change_quantity(
    $item['item_id'],
    $item['item_location'],
    $item['quantity_purchased'] * (-$item['receiving_quantity'])  // 负号表示减少
);
```

### 2.4 金额计算中的使用

**代码位置**: [Receiving_lib.php::get_item_total()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Libraries/Receiving_lib.php#L485-L497)

```php
public function get_item_total(float $quantity, float $price, float $discount, ?int $discount_type, float $receiving_quantity): string
{
    $extended_quantity = bcmul($quantity, $receiving_quantity);  // 实际数量 = 采购数 × 倍率
    $total = bcmul($extended_quantity, $price);                   // 总金额 = 实际数量 × 单价
    // ... 折扣处理
}
```

### 2.5 receiving_quantity 对销售的影响 —— **无影响**

**关键发现**: 销售出库时 **完全不使用** `receiving_quantity`。

**代码位置**: [Sale.php::save_value()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Models/Sale.php#L635-L662)

```php
if ($cur_item_info->stock_type == HAS_STOCK && $sale_status == COMPLETED) {
    // 扣减库存：直接使用 quantity，不经过任何倍率换算
    $item_quantity->save_value([
        'quantity' => $item_quantity_data->quantity - $item_data['quantity'],  // 无换算
        // ...
    ]);
    
    // 流水记录：直接使用 -quantity
    $inv_data = [
        'trans_inventory' => -$item_data['quantity']  // 无换算
    ];
}
```

**结论**: `receiving_quantity` **仅作用于收货模块**，销售模块不使用该字段。销售时的包装换算使用 `qty_per_pack`（销售包装）。

---

## 3. 六大库存操作的完整代码路径与一致性分析

### 3.1 操作总览：各场景的检查与执行对比

| 场景 | 检查 stock_type | 检查 item_type | receiving_quantity 参与 | 事务保护 |
|------|----------------|----------------|------------------------|----------|
| **商品保存 postSave()** | ❌ 不检查 | ✅ 仅 ITEM_TEMP 强制清库存 | ❌ | ❌ 无 |
| **库存调整 postSaveInventory()** | ❌ 完全不检查 | ❌ 完全不检查 | ❌ | ❌ 无 |
| **销售出库 Sale::save_value()** | ✅ `HAS_STOCK` 才扣减 | ❌ 不检查 | ❌ | ✅ 有 |
| **收货入库 Receiving::save_value()** | ❌ 完全不检查 | ❌ 完全不检查 | ✅ 参与计算 | ✅ 有 |
| **CSV 导入 save_inventory_quantities()** | ❌ 完全不检查 | ❌ 完全不检查 | ❌ | ✅ 有（外部） |
| **删除收货 Receiving::delete_value()** | ❌ 完全不检查 | ❌ 完全不检查 | ✅ 参与计算 | ✅ 有 |

---

### 3.2 商品保存时调整库存 (postSave)

**控制器**: [Items.php::postSave()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Controllers/Items.php#L618-L759)

#### 3.2.1 ITEM_TEMP 的特殊处理

```php
// Lines 716-718: 临时商品强制库存为0
if ($item_data['item_type'] == ITEM_TEMP) {
    $updated_quantity = 0;
}
```

#### 3.2.2 库存更新逻辑

```php
foreach ($stock_locations as $location) {
    $updated_quantity = parse_quantity($this->request->getPost('quantity_' . $location['location_id']));

    if ($item_data['item_type'] == ITEM_TEMP) {
        $updated_quantity = 0;   // ITEM_TEMP 强制清零
    }

    $item_quantity = $this->item_quantity->get_item_quantity($item_id, $location['location_id']);

    if ($item_quantity->quantity != $updated_quantity || $new_item) {
        // 1. 更新 item_quantities（先）
        $success = $success && $this->item_quantity->save_value($location_detail, $item_id, $location['location_id']);

        // 2. 插入 inventory 流水（后）—— 存差值
        $inv_data = [
            'trans_comment'   => lang('Items.manually_editing_of_quantity'),
            'trans_inventory' => $updated_quantity - $item_quantity->quantity
        ];
        $success = $success && $this->inventory->insert($inv_data, false);
    }
}
```

#### 3.2.3 问题

- **❌ 未检查 stock_type**：即使商品是 `HAS_NO_STOCK`，用户输入了库存数量照样写入 `item_quantities` 和 `inventory`
- **❌ 无事务保护**：两个操作可能部分成功
- **✅ 仅 ITEM_TEMP 特殊处理**：库存强制清零

---

### 3.3 专门的库存调整表单 (postSaveInventory)

**控制器**: [Items.php::postSaveInventory()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Controllers/Items.php#L855-L889)

```php
// ⚠️ 完全没有任何 stock_type / item_type 检查！
$cur_item_info = $this->item->get_info($item_id);  // 获取了商品信息但未使用

// 1. 先插入流水
$inv_data = [
    'trans_inventory' => parse_quantity($new_quantity)  // 直接存输入的增减量
];
$this->inventory->insert($inv_data, false);

// 2. 后更新数量
$item_quantity_data = [
    'quantity' => $item_quantity->quantity + parse_quantity($this->request->getPost('newquantity'))
];
$this->item_quantity->save_value($item_quantity_data, $item_id, $location_id);
```

#### 问题

- **❌ 完全不检查 stock_type**：`HAS_NO_STOCK` / `ITEM_TEMP` 商品照样可以调整库存
- **❌ 完全不检查 item_type**
- **❌ 操作顺序与 postSave 相反**：先插流水后更数量
- **❌ 无事务保护**
- **trans_inventory 含义不同**：存的是「增量」而非「目标值-当前值」的差值（与 postSave 不同但数学等价）

---

### 3.4 销售出库 (Sale::save_value)

**模型**: [Sale.php::save_value()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Models/Sale.php#L518-L688)

```php
// ✅ 唯一有 stock_type 检查的场景！
if ($cur_item_info->stock_type == HAS_STOCK && $sale_status == COMPLETED) {
    // 更新 item_quantities（先）
    $item_quantity->save_value([
        'quantity'    => $item_quantity_data->quantity - $item_data['quantity'],
        // ...
    ]);

    // 插入 inventory 流水（后）
    $inv_data = [
        'trans_comment'   => 'POS ' . $sale_id,
        'trans_inventory' => -$item_data['quantity']
    ];
    $inventory->insert($inv_data, false);
}
```

#### 特点

- **✅ 检查 stock_type**：只有 `HAS_STOCK` 才扣库存和写流水
- **❌ 不检查 item_type**：理论上 `ITEM_TEMP` 商品被强制 `HAS_NO_STOCK`，所以间接被挡住了
- **✅ 事务保护**：在 `transStart` / `transComplete` 内
- **receiving_quantity 不参与计算**：销售扣减与收货倍率无关

---

### 3.5 收货入库 (Receiving::save_value)

**模型**: [Receiving.php::save_value()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Models/Receiving.php#L103-L190)

```php
// ❌ 完全不检查 stock_type！
// 无论 HAS_STOCK 还是 HAS_NO_STOCK，都写入库存！

$items_received = $item_data['receiving_quantity'] != 0
    ? $item_data['quantity'] * $item_data['receiving_quantity']
    : $item_data['quantity'];

// 1. 更新数量（先）
$item_quantity->save_value([
    'quantity' => $item_quantity_value->quantity + $items_received,
    // ...
]);

// 2. 插入流水（后）
$inv_data = [
    'trans_comment'   => 'RECV ' . $receiving_id,
    'trans_inventory' => $items_received
];
$inventory->insert($inv_data, false);
```

#### 问题

- **❌ 完全不检查 stock_type**：`HAS_NO_STOCK` 商品（服务类、临时商品）也能入库并产生流水
- **❌ 完全不检查 item_type**：`ITEM_TEMP` 商品被强制 `receiving_quantity=0`，所以 `items_received = quantity * 0 = 0`，实际不会改变库存数量，但仍然会 **插入一条 trans_inventory=0 的流水记录**
- **✅ 事务保护**
- **✅ receiving_quantity 参与计算**

#### ITEM_TEMP 收货时的实际行为推演

```
ITEM_TEMP 商品：
  stock_type = HAS_NO_STOCK  （强制）
  receiving_quantity = 0     （强制）
  
收货时：
  items_received = quantity * 0 = 0
  item_quantities += 0   → 库存不变
  inventory 插入一条 trans_inventory=0 的流水 ✗（产生了无意义记录）
```

---

### 3.6 CSV 导入 (save_inventory_quantities)

**控制器**: [Items.php::postImportCsvFile()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Controllers/Items.php#L979-L1107)

**内部方法**: [Items.php::save_inventory_quantities()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Controllers/Items.php#L1264-L1299)

#### 3.6.1 itemData 构建时的缺失

**代码 Lines 1013-1024**:

```php
$itemData = [
    'item_id'       => $itemId,
    'name'          => $row['Item Name'],
    'description'   => filter_var($row['Description'], ...),
    'category'      => $row['Category'],
    'cost_price'    => $row['Cost Price'],
    'unit_price'    => $row['Unit Price'],
    'reorder_level' => $row['Reorder Level'],
    'deleted'       => false,
    'hsn_code'      => $row['HSN'],
    'pic_filename'  => $row['Image']
];
// ❌ 未设置 stock_type → 使用数据库默认值（通常为 0=HAS_STOCK）
// ❌ 未设置 item_type → 使用数据库默认值（通常为 0=ITEM）
// ❌ 未设置 receiving_quantity → 使用数据库默认值
```

**CSV 模板中 stock_type / item_type 的映射**：需要检查 CSV 模板是否包含这些字段。根据代码，这些字段不在 `$itemData` 中显式设置。

#### 3.6.2 库存写入逻辑

```php
private function save_inventory_quantities(array $row, array $item_data, array $allowed_locations, int $employee_id): bool
{
    foreach ($allowed_locations as $location_id => $location_name) {
        $csv_data = [
            'trans_items'    => $item_data['item_id'],
            'trans_user'     => $employee_id,
            'trans_comment'  => lang('Items.inventory_CSV_import_quantity'),
            'trans_location' => $location_id
        ];

        if (!empty($row["location_$location_name"]) || $row["location_$location_name"] === '0') {
            // 有明确数量的情况
            $item_quantity_data['quantity'] = $row["location_$location_name"];
            $success &= $this->item_quantity->save_value($item_quantity_data, ...);

            $csv_data['trans_inventory'] = $row["location_$location_name"];  // ⚠️ 存的是绝对值！
            $success &= (bool)$this->inventory->insert($csv_data, false);
            
        } elseif ($is_update) {
            // 更新已有商品：CSV 该列为空 → 跳过，不修改库存
            continue;
            
        } else {
            // 新建商品：CSV 该列为空 → 初始化为 0
            $item_quantity_data['quantity'] = 0;
            $success &= $this->item_quantity->save_value($item_quantity_data, ...);

            $csv_data['trans_inventory'] = 0;
            $success &= (bool)$this->inventory->insert($csv_data, false);
        }
    }
}
```

#### 3.6.3 重大问题：trans_inventory 存储绝对值而非增量

```
假设商品原库存：50
CSV 中填写：80

实际操作：
  item_quantities 设置为 80  ✅（正确）
  inventory 插入 trans_inventory = 80 ❌（应该是 80-50=30）

一致性校验公式失效：
  SUM(trans_inventory) = 80 + 之前的流水
  item_quantities = 80
  → 两者不相等！
```

**这是 CSV 导入时库存流水与实际数量不一致的根本原因。**

#### 3.6.4 其他问题

- **❌ 完全不检查 stock_type**：任何商品都可以通过 CSV 导入库存
- **❌ 完全不检查 item_type**
- **❌ 新商品时 itemData 缺少 stock_type/item_type**：使用数据库默认值，可能与预期不符
- **✅ 外层有事务保护**：`$db->transBegin()` / `transCommit()` 包裹整个 CSV 导入循环

---

## 4. 各操作中商品类型矩阵行为分析

### 4.1 ITEM + HAS_STOCK（标准库存商品）

| 操作 | item_quantities | inventory 流水 | 说明 |
|------|----------------|---------------|------|
| postSave 调整 | ✅ 更新 | ✅ 记录差值 | 正常 |
| postSaveInventory | ✅ 更新 | ✅ 记录增量 | 正常 |
| 销售出库 | ✅ 扣减 | ✅ 记录负值 | 正常 |
| 收货入库 | ✅ 增加（×倍率） | ✅ 记录（×倍率） | 正常 |
| CSV 导入 | ✅ 设置 | ❌ 记录绝对值 | **不一致！** |
| 删除收货 | ✅ 回滚（×倍率） | ✅ 回滚（×倍率） | 正常 |

### 4.2 ITEM + HAS_NO_STOCK（无库存服务商品）

| 操作 | item_quantities | inventory 流水 | 说明 |
|------|----------------|---------------|------|
| postSave 调整 | ✅ 仍可写入 | ✅ 仍可记录 | **逻辑矛盾：既然是无库存，为什么还能写？** |
| postSaveInventory | ✅ 仍可写入 | ✅ 仍可记录 | **逻辑矛盾** |
| 销售出库 | ❌ 不操作 | ❌ 不操作 | 正常（stock_type 拦截） |
| 收货入库 | ✅ 仍可写入 | ✅ 仍可记录 | **逻辑矛盾** |
| CSV 导入 | ✅ 仍可写入 | ❌ 记录绝对值 | **逻辑矛盾 + 不一致** |
| 删除收货 | ✅ 仍可回滚 | ✅ 仍可回滚 | 正常（反向操作抵消） |

### 4.3 ITEM_TEMP（临时商品）

| 操作 | item_quantities | inventory 流水 | 说明 |
|------|----------------|---------------|------|
| postSave 调整 | ✅ 强制清零 | ✅ 记录（清0-原值） | 强制约束生效 |
| postSaveInventory | ✅ 仍可写入 | ✅ 仍可记录 | **无拦截！用户可绕过强制清零** |
| 销售出库 | ❌ 不操作 | ❌ 不操作 | 正常（因强制 HAS_NO_STOCK） |
| 收货入库 | ✅ 不变（×0） | ✅ 记录 0 值 | 产生无意义流水 |
| CSV 导入 | ✅ 仍可写入 | ❌ 记录绝对值 | **无拦截** |
| 删除收货 | ✅ 不变（×0） | ✅ 记录 0 值 | 产生无意义流水 |

### 4.4 ITEM_KIT（套装商品）

| 操作 | item_quantities | inventory 流水 | 说明 |
|------|----------------|---------------|------|
| 销售套装 | ❌ 套装本身不扣 | ❌ 套装本身不记 | 正确（通过子商品扣减） |
| 收货入库 | ✅ 套装也入库 | ✅ 套装也记流水 | **通常不合逻辑** |

---

## 5. 负库存边界检查

### 5.1 唯一的检查点：Sale_lib::out_of_stock()

**位置**: [Sale_lib.php::out_of_stock()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Libraries/Sale_lib.php#L1193-L1212)

```php
public function out_of_stock(int $item_id, int $item_location): string
{
    if ($item_id != -1) {
        $item_info = $this->item->get_info_by_id_or_number($item_id);
        
        // 只检查 HAS_STOCK 商品
        if ($item_info->stock_type == HAS_STOCK) {
            $item_quantity = $this->item_quantity->get_item_quantity($item_id, $item_location)->quantity;
            $quantity_added = $this->get_quantity_already_added($item_id, $item_location);
            
            if ($item_quantity - $quantity_added < 0) {
                return lang('Sales.quantity_less_than_zero');    // 库存不足
            } elseif ($item_quantity - $quantity_added < $item_info->reorder_level) {
                return lang('Sales.quantity_less_than_reorder_level');  // 低于预警
            }
        }
    }
    return '';
}
```

### 5.2 检查时机 —— 仅在加入购物车时

| 操作 | 触发检查的位置 | 效果 |
|------|--------------|------|
| 添加单品到购物车 | [Sale_lib.php::add_item()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Libraries/Sale_lib.php) 返回的 cart 中包含 `out_of_stock` 字段 | 前端显示警告，**不阻止** |
| 添加套装到购物车 | [Sale_lib.php::add_item_kit()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Libraries/Sale_lib.php) 对子商品逐个检查 | 同上 |
| 完成销售扣减库存 | **无检查** | 直接扣减，可能变负数 |
| 手动调整库存 | **无检查** | 可随意设置为负值 |
| CSV 导入库存 | **无检查** | 可随意设置为负值 |

### 5.3 负库存可能出现的路径

```
路径1：并发销售（最常见）
  收银员A：检查库存 → 有货 10
  收银员B：检查库存 → 有货 10（同一时间）
  收银员A：完成销售 → 扣减 10 → 库存 0
  收银员B：完成销售 → 扣减 10 → 库存 -10 ❌

路径2：postSaveInventory 直接输入负数
  用户直接输入 newquantity = -50
  → 库存直接减少 50 → 可能为负 ❌

路径3：postSave 手动输入目标值为负
  用户在 quantity_1 输入 -10
  → item_quantities = -10
  → 流水记录 -10 - 原值 ❌

路径4：CSV 导入负值
  location_仓库1 填 -20
  → 库存直接设为 -20 ❌

路径5：删除收货（正常回滚但可能超出现有库存）
  收货入库 100 → 销售出库 80 → 库存 20
  删除该收货单 → 回滚 -100 → 库存 20 - 100 = -80 ❌
  （但这是业务上合理的反向操作，是否允许需业务决策）
```

---

## 6. 一致性问题汇总与风险评级

### 6.1 数据不一致风险矩阵

| 风险点 | 影响范围 | 严重程度 | 发生概率 |
|--------|---------|---------|---------|
| CSV 导入 trans_inventory 存绝对值 | 所有通过 CSV 导入的商品 | 🔴 高 | 🔴 高 |
| postSave / postSaveInventory 操作顺序相反 | 所有手动调整商品 | 🟡 中 | 🟡 中 |
| postSave / postSaveInventory 无事务 | 所有手动调整商品 | 🟡 中 | 🟠 低（但可能发生） |
| 删除收货无 stock_type 检查 | HAS_NO_STOCK 商品的收货 | 🟡 中 | 🟠 低 |
| 收货入库无 stock_type 检查 | HAS_NO_STOCK 商品的收货 | 🟡 中 | 🟡 中 |
| postSaveInventory 无类型检查 | ITEM_TEMP / HAS_NO_STOCK | 🟡 中 | 🟡 中 |
| ITEM_TEMP 收货产生 0 值流水 | ITEM_TEMP 商品的收货 | 🟢 低 | 🟡 中 |
| 销售实际扣减无二次库存检查 | 高并发销售场景 | 🔴 高 | 🟡 中 |

### 6.2 一致性校验 SQL

```sql
-- 1. 库存数量 vs 流水汇总 一致性检查（核心）
SELECT 
    iq.item_id,
    i.name,
    i.stock_type,
    i.item_type,
    iq.location_id,
    sl.location_name,
    iq.quantity AS current_quantity,
    COALESCE(SUM(inv.trans_inventory), 0) AS inventory_sum,
    iq.quantity - COALESCE(SUM(inv.trans_inventory), 0) AS diff
FROM item_quantities iq
JOIN items i ON i.item_id = iq.item_id
LEFT JOIN stock_locations sl ON sl.location_id = iq.location_id
LEFT JOIN inventory inv ON inv.trans_items = iq.item_id AND inv.trans_location = iq.location_id
GROUP BY iq.item_id, iq.location_id
HAVING diff != 0
ORDER BY ABS(diff) DESC;

-- 2. 找出 HAS_NO_STOCK 但有库存数据的异常商品
SELECT 
    i.item_id,
    i.name,
    i.stock_type,
    i.item_type,
    iq.location_id,
    iq.quantity
FROM items i
JOIN item_quantities iq ON iq.item_id = i.item_id
WHERE i.stock_type = 1  -- HAS_NO_STOCK
  AND iq.quantity != 0;

-- 3. 找出 ITEM_TEMP 但有库存数据的异常商品
SELECT 
    i.item_id,
    i.name,
    i.item_type,
    i.stock_type,
    iq.location_id,
    iq.quantity
FROM items i
JOIN item_quantities iq ON iq.item_id = i.item_id
WHERE i.item_type = 3  -- ITEM_TEMP
  AND iq.quantity != 0;

-- 4. 找出 inventory 表中 trans_inventory=0 的无意义记录
SELECT 
    trans_id,
    trans_items,
    trans_location,
    trans_date,
    trans_comment
FROM inventory
WHERE trans_inventory = 0;
```

---

## 7. 改进建议（按优先级排序）

### 7.1 紧急修复（高优先级）

**① CSV 导入：trans_inventory 改为存储增量而非绝对值**

```php
// 修改 save_inventory_quantities()
if (!empty($row["location_$location_name"]) || $row["location_$location_name"] === '0') {
    $new_quantity = $row["location_$location_name"];
    $old_quantity = $this->item_quantity->get_item_quantity($item_data['item_id'], $location_id)->quantity;
    
    $item_quantity_data['quantity'] = $new_quantity;
    $success &= $this->item_quantity->save_value($item_quantity_data, ...);
    
    // ✅ 存差值，不是绝对值
    $csv_data['trans_inventory'] = $new_quantity - $old_quantity;
    $success &= (bool)$this->inventory->insert($csv_data, false);
}
```

**② 销售扣减时增加二次库存检查（防并发）**

在 `Sale.php::save_value()` 实际扣减前增加检查，或使用数据库行锁。

### 7.2 重要修复（中优先级）

**③ 统一 stock_type 检查，所有入库操作加拦截**

```php
// 在 Receiving.php::save_value() 收货入库前增加
if ($cur_item_info->stock_type != HAS_STOCK) {
    continue;  // 跳过非库存商品的收货入库处理
}
```

```php
// 在 Items.php::postSaveInventory() 调整库存前增加
if ($cur_item_info->stock_type != HAS_STOCK || $cur_item_info->item_type == ITEM_TEMP) {
    return $this->response->setJSON(['success' => false, 'message' => lang('Items.item_not_stock_type')]);
}
```

**④ postSave 和 postSaveInventory 增加事务包裹**

**⑤ 统一操作顺序**：建议所有场景采用「先插流水 → 后更数量」或统一相反，不再混用

### 7.3 优化修复（低优先级）

**⑥ 删除 CSV 导入中新建商品时 trans_inventory=0 的记录**：创建商品时初始化库存为0无需记录流水，除非有实际变化

**⑦ 删除 ITEM_TEMP 收货时产生的 0 值流水**：增加判断 `if ($items_received != 0)` 才插入流水

**⑧ delete_value 与 save_value 统一 receiving_quantity=0 的处理逻辑**（详见 9.2 节的不一致 Bug）

---

## 9. 临时商品/无库存商品在收货链路的代码级深度分析

### 9.1 为什么收货入口没有 stock_type 拦截？—— 代码设计溯源

#### 9.1.1 销售 vs 收货的不对称设计

**Sale_lib（销售链路）**:
```php
// Sale_lib.php 中有完整的 stock_type 检查
public function out_of_stock(int $item_id, int $item_location): string
{
    // ...
    if ($item_info->stock_type == HAS_STOCK) {
        // 只对 HAS_STOCK 商品做库存检查
    }
}

// Sale.php::save_value() 中有最终拦截
if ($cur_item_info->stock_type == HAS_STOCK && $sale_status == COMPLETED) {
    // 只有 HAS_STOCK 才扣库存
}
```

**Receiving_lib（收货链路）**:
```php
// ❌ Receiving_lib.php 中完全没有 stock_type 相关代码！
// Grep 结果：0 处匹配 HAS_STOCK / HAS_NO_STOCK / stock_type
```

**Receiving.php::save_value()**:
```php
foreach ($items as $line => $item_data) {
    $cur_item_info = $item->get_info($item_data['item_id']);
    // 已经 $cur_item_info 拿出来了，但完全没检查 stock_type！
    // ↓↓↓ 直接走下去 ↓↓↓
    $items_received = $item_data['receiving_quantity'] != 0
        ? $item_data['quantity'] * $item_data['receiving_quantity']
        : $item_data['quantity'];
    // 更新库存...
    // 插入流水...
}
```

#### 9.1.2 根本原因推测

从迁移脚本 [20170501000000_initial_schema.php](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Database/Migrations/20170501000000_initial_schema.php) 和早期 SQL 迁移可以看出：

1. **`receiving_quantity` 是后来加的字段**：`phppos_migrate.sql` 第79行显示，从旧版本迁移时 `receiving_quantity` 被硬编码为 `1`
2. **`stock_type`/`item_type` 是更晚才加入的概念**：引入时只在销售侧加了拦截，收货侧遗漏了
3. **ITEM_TEMP 是最新功能**：`postSave()` 中对 ITEM_TEMP 做了强制约束，但这种约束没有「扩散」到收货/库存调整等其他入口

### 9.2 receiving_quantity=0 时：save_value 与 delete_value 的计算逻辑不一致（严重 Bug）

#### 9.2.1 save_value 中的处理（入库时）

**代码**: [Receiving.php::save_value()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Models/Receiving.php#L154)

```php
$items_received = $item_data['receiving_quantity'] != 0
    ? $item_data['quantity'] * $item_data['receiving_quantity']
    : $item_data['quantity'];    // ✅ receiving_quantity=0 时，退化为 items_received = quantity
```

#### 9.2.2 delete_value 中的处理（删除/回滚时）

**代码**: [Receiving.php::delete_value()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Models/Receiving.php#L238-L244)

```php
$trans_inventory = $item['quantity_purchased'] * (-$item['receiving_quantity']);
// ❌ receiving_quantity=0 时，直接 = 0，不会退化为 quantity！

$item_quantity->change_quantity(
    $item['item_id'],
    $item['item_location'],
    $item['quantity_purchased'] * (-$item['receiving_quantity'])  // ❌ 同样 = 0
);
```

#### 9.2.3 推演场景：普通商品（非 ITEM_TEMP）被手动设为 receiving_quantity=0

| 步骤 | 操作 | item_quantities 变化 | inventory 流水记录 |
|------|------|---------------------|-------------------|
| 初始 | 库存 = 100 | 100 | SUM = 100 |
| ① | 收货：quantity=10, receiving_quantity=0 | 100 + **10** = 110 ✅（退化逻辑） | RECV x → **trans_inventory = 10** ✅ |
| ② | 删除该收货单 | 110 + **0** = 110 ❌（不会退化为 -10） | Deleting receiving x → **trans_inventory = 0** ❌ |
| 最终 | | **110**（应该是 100） | **SUM = 110**（流水多了 10） |

**结果**：
- `item_quantities` 永久多了 10 个库存
- `inventory` 流水总和也永久多了 10（两者相等，所以一致性校验查不出来！）
- **这是「幽灵库存」产生的原因之一**

#### 9.2.4 推演场景：ITEM_TEMP 商品（强制 receiving_quantity=0）

| 步骤 | 操作 | item_quantities 变化 | inventory 流水记录 |
|------|------|---------------------|-------------------|
| 初始 | 库存 = 0（强制） | 0 | SUM = 0 |
| ① | 收货：quantity=100, **receiving_quantity=0**（强制） | 0 + **100** = 100 ❌（退化逻辑生效了！） | RECV x → **trans_inventory = 100** ❌ |
| ② | 删除该收货单 | 100 + **0** = 100 ❌❌ | Deleting receiving x → **trans_inventory = 0** ❌ |
| 最终 | | **100**（永久幽灵库存！） | **SUM = 100**（一致性校验通过但业务上错误） |

**双重 Bug**：
1. ITEM_TEMP 强制 `receiving_quantity=0` 但 **入库时退化逻辑生效**，库存照样增加（与 postSave 中 ITEM_TEMP 强制清 0 的意图完全违背）
2. 删除时退化逻辑 **不生效**，导致无法抵消
3. 结果：ITEM_TEMP 商品收货后再删除 → **永久产生 100 个幽灵库存**

### 9.3 ITEM_TEMP / HAS_NO_STOCK 在收货链路的完整代码执行路径

#### 9.3.1 入库时（Receiving::save_value）

```
输入：item_id（ITEM_TEMP 商品，stock_type=HAS_NO_STOCK，receiving_quantity=0）
     quantity = 50

执行路径：
  1. $cur_item_info = $item->get_info($item_data['item_id'])
     → 拿到了 stock_type=HAS_NO_STOCK，但 ⚠️ 没有任何 if 判断
     
  2. $builder->insert($receivings_items_data)
     → receivings_items 表写入：
       quantity_purchased = 50
       receiving_quantity = 0  ← 存入了 0
       
  3. $items_received = receiving_quantity != 0 ? quantity * 0 : quantity
     → = 0 != 0 ? ... : 50
     → = 50  ⚠️ ITEM_TEMP 的 receiving_quantity=0 反而触发了退化逻辑！
     
  4. change_cost_price($item_id, items_received=50, ...)
     → 如果配置了 receiving_calculate_average_price=1
     → 即使是 ITEM_TEMP，成本价也会被重算 ❌
     
  5. item_quantities.quantity += 50
     → ITEM_TEMP 商品库存变成 50 ❌（违背 ITEM_TEMP 不应有库存的设计初衷）
     
  6. inventory 插入 trans_inventory = 50
     → 产生了真实流水记录 ❌
```

#### 9.3.2 删除收货时（Receiving::delete_value）

```
输入：receiving_id = 上面那张单，update_inventory = true

执行路径：
  1. $items = get_receiving_items($receiving_id)
     → 从 receivings_items 读出：
       quantity_purchased = 50
       receiving_quantity = 0
       
  2. 遍历 $items（同样 ⚠️ 没有 stock_type 判断）
  
  3. inventory 插入流水：
     trans_inventory = 50 * (-0) = 0
     → ⚠️ 插入了一条 trans_inventory=0 的无意义记录（与 save_value 的 50 不匹配！）
     
  4. change_quantity($item_id, $location_id, 50 * (-0) = 0)
     → 库存不变，仍然是 50 ❌❌
     → 入库的 50 无法被抵消！
```

### 9.4 「receiving_quantity=0 退化」与「ITEM_TEMP 强制清零」的设计冲突总结

| 设计意图 | 实现位置 | 是否生效 |
|---------|---------|---------|
| ITEM_TEMP 商品不参与库存管理 | Items.php::postSave() → 强制 stock_type=HAS_NO_STOCK | 销售侧生效 |
| ITEM_TEMP 商品 receiving_quantity=0 | Items.php::postSave() → 强制设 0 | 保存商品时生效，但 **收货侧反而触发退化** |
| ITEM_TEMP 商品库存强制为 0 | Items.php::postSave() → updated_quantity=0 | 仅通过 postSave 入口生效，其他入口绕过 |
| receiving_quantity=0 时退化为 1 | Receiving.php::save_value() → 三元运算符 | 任何商品都生效，**包括 ITEM_TEMP** ❌ |

**根本矛盾**：`postSave()` 想通过「设 receiving_quantity=0」来表达 ITEM_TEMP 不参与收货入库，但 `save_value()` 中三元运算符的退化设计刚好把 `=0` 理解为「不需要换算，直接用 quantity」，两者语义完全相反。

### 9.5 为何一致性校验 SQL 查不出幽灵库存？

```sql
-- 第 6 节给出的一致性校验 SQL
HAVING iq.quantity - COALESCE(SUM(inv.trans_inventory), 0) != 0
```

对于上述 ITEM_TEMP 场景：
- `item_quantities.quantity = 100`（幽灵库存）
- `SUM(trans_inventory) = 100`（入库 100 + 删除回滚 0 = 100）
- **差值 = 0 → 校验通过！**

因为 save_value 和 delete_value 虽然逻辑相反、数量不匹配，但两者都「同时污染」了两张表，所以表间校验 SQL 查不出问题。需要结合 **业务规则校验** 才能发现：

```sql
-- 补充 SQL：查出 HAS_NO_STOCK 或 ITEM_TEMP 但有库存的商品
SELECT i.item_id, i.name, i.item_type, i.stock_type, iq.location_id, iq.quantity
FROM items i
JOIN item_quantities iq ON iq.item_id = i.item_id
WHERE (i.stock_type = 1 OR i.item_type = 3)
  AND iq.quantity != 0;
```

---

## 10. 附录：数据流全景图（补充）

### 10.1 收货完整链路（含 receiving_quantity，标注 Bug 点）

```
用户添加商品到收货车
    ↓
Receiving_lib::add_item()
  └─ ❌ 无 stock_type 检查，任何商品都能加入
  └─ 读取商品的 receiving_quantity 作为默认值
  └─ 存入 recv_cart session 数组
    ↓
用户完成收货，点击提交
    ↓
Receivings::postComplete()
  └─ Receiving_lib::get_cart() 获取购物车数据
  └─ Receiving::save_value() 执行入库（事务内）
      ├─ 插入 receivings_items: receiving_quantity 原样存入
      ├─ items_received = receiving_quantity!=0 ? qty*rq : qty
      │   └─ ⚠️ ITEM_TEMP 因 rq=0 触发退化 → 实际入库 qty
      ├─ 如果配置 receiving_calculate_average_price
      │   └─ Item::change_cost_price() → 重算成本价
      ├─ item_quantities += items_received
      │   └─ ❌ 无 stock_type 检查，HAS_NO_STOCK 也增加
      ├─ inventory: trans_inventory = items_received
      │   └─ ❌ 无判断，即使 items_received=0 也插入
      └─ attribute copy_attribute_links
    ↓
删除该收货单
    ↓
Receiving::delete_value()（事务内）
  └─ ❌ 无 stock_type 检查
  ├─ inventory: trans_inventory = qty_purchased × (-rq)
  │   └─ ⚠️ rq=0 时 =0，与入库时的 qty 不匹配 → 不一致
  └─ change_quantity: change = qty_purchased × (-rq)
      └─ ⚠️ rq=0 时 =0，库存没被回滚 → 幽灵库存产生
```

### 8.1 收货完整链路（含 receiving_quantity）

```
用户添加商品到收货车
    ↓
Receiving_lib::add_item()
  └─ 读取商品的 receiving_quantity 作为默认值（可修改）
  └─ 存入 recv_cart session 数组
    ↓
用户完成收货，点击提交
    ↓
Receivings::postComplete()
  └─ Receiving_lib::get_cart() 获取购物车数据
  └─ Receiving::save_value() 执行入库
      ├─ items_received = quantity × receiving_quantity
      ├─ 更新 item_quantities += items_received
      ├─ 插入 inventory 流水: trans_inventory = items_received
      └─ 写入 receivings / receivings_items 表
```

### 8.2 商品创建/编辑链路

```
postSave()
  ├─ 处理 item_type
  │   ├─ ITEM_TEMP → 强制 stock_type=HAS_NO_STOCK, receiving_quantity=0
  │   └─ 其他类型 → receiving_quantity=0? → 强制为 1
  ├─ 保存 items 表
  └─ 遍历仓库位置更新库存
      ├─ ITEM_TEMP → updated_quantity = 0
      ├─ 比较新旧数量
      │   ├─ 更新 item_quantities（先）
      │   └─ 插入 inventory 流水（后）: 存差值
      └─ ⚠️ 无事务，不检查 stock_type
```

### 8.3 销售出库链路

```
Sale_lib::add_item() → out_of_stock() 检查（仅警告）
    ↓
完成销售 Sales::postComplete()
    ↓
Sale::save_value()
  └─ transStart() 开启事务
  └─ 遍历购物车
      ├─ 检查 stock_type == HAS_STOCK && sale_status == COMPLETED
      │   ├─ 更新 item_quantities -= quantity（先）
      │   ├─ 插入 inventory: trans_inventory = -quantity（后）
      │   └─ ⚠️ 无二次库存检查
      └─ transComplete() 提交事务
```
