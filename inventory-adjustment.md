# 商品数量调整与库存流水一致性分析

## 1. 核心数据结构

### 1.1 商品资料 (items 表)

**模型文件**: [Item.php](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Models/Item.php)

| 字段 | 类型 | 说明 | 常量定义 |
|------|------|------|----------|
| `stock_type` | int | 库存类型 | `HAS_STOCK` = 0 (有库存), `HAS_NO_STOCK` = 1 (无库存) |
| `item_type` | int | 商品类型 | `ITEM` = 0, `ITEM_KIT` = 1, `ITEM_AMOUNT_ENTRY` = 2, `ITEM_TEMP` = 3 |
| `reorder_level` | decimal | 库存预警水平 | - |
| `receiving_quantity` | decimal | 默认收货数量 | - |
| `qty_per_pack` | decimal | 每包数量 | - |

**关键常量定义** ([Constants.php](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Config/Constants.php#L107-L114)):

```php
const HAS_STOCK = 0;      // 有库存商品
const HAS_NO_STOCK = 1;   // 无库存商品（服务类等）
const ITEM = 0;           // 普通商品
const ITEM_KIT = 1;       // 商品套装
const ITEM_AMOUNT_ENTRY = 2;  // 金额输入商品
const ITEM_TEMP = 3;      // 临时商品
```

### 1.2 库存数量 (item_quantities 表)

**模型文件**: [Item_quantity.php](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Models/Item_quantity.php)

| 字段 | 类型 | 说明 |
|------|------|------|
| `item_id` | int | 商品ID (主键) |
| `location_id` | int | 仓库位置ID (主键) |
| `quantity` | decimal | 当前库存数量 |

**核心方法** ([Item_quantity.php#L91-L98](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Models/Item_quantity.php#L91-L98)):

```php
public function change_quantity(int $item_id, int $location_id, int $quantity_change): bool
{
    $quantity_old = $this->get_item_quantity($item_id, $location_id);
    $quantity_new = $quantity_old->quantity + $quantity_change;
    $location_detail = ['item_id' => $item_id, 'location_id' => $location_id, 'quantity' => $quantity_new];
    return $this->save_value($location_detail, $item_id, $location_id);
}
```

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

## 2. 库存增减的代码路径

系统中有 **5个主要入口** 会触发库存变化，所有入口都会同时更新 `item_quantities`（库存数量）和 `inventory`（库存流水）。

### 2.1 商品保存时调整库存 (手动调整)

**控制器**: [Items.php::postSave()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Controllers/Items.php#L618-L759)

```php
// 代码片段 (Lines 712-742)
foreach ($stock_locations as $location) {
    $updated_quantity = parse_quantity($this->request->getPost('quantity_' . $location['location_id']));
    
    $item_quantity = $this->item_quantity->get_item_quantity($item_id, $location['location_id']);
    
    if ($item_quantity->quantity != $updated_quantity || $new_item) {
        // 1. 更新 item_quantities 表
        $success = $success && $this->item_quantity->save_value($location_detail, $item_id, $location['location_id']);
        
        // 2. 插入 inventory 流水记录
        $inv_data = [
            'trans_date'      => date('Y-m-d H:i:s'),
            'trans_items'     => $item_id,
            'trans_user'      => $employee_id,
            'trans_location'  => $location['location_id'],
            'trans_comment'   => lang('Items.manually_editing_of_quantity'),
            'trans_inventory' => $updated_quantity - $item_quantity->quantity  // 差值
        ];
        $success = $success && $this->inventory->insert($inv_data, false);
    }
}
```

**特点**:
- 先更新 `item_quantities`，后插入 `inventory`
- `trans_inventory` 存储的是 **变化差值** (新数量 - 旧数量)
- 两个操作在同一个循环内，无数据库事务包裹

### 2.2 专门的库存调整表单

**控制器**: [Items.php::postSaveInventory()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Controllers/Items.php#L855-L889)

**视图**: [form_inventory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Views/items/form_inventory.php)

```php
// 代码片段 (Lines 861-880)
$inv_data = [
    'trans_date'      => date('Y-m-d H:i:s'),
    'trans_items'     => $item_id,
    'trans_user'      => $employee_id,
    'trans_location'  => $location_id,
    'trans_comment'   => $this->request->getPost('trans_comment'),
    'trans_inventory' => parse_quantity($new_quantity)  // 直接存储输入的增减量
];

// 1. 先插入 inventory 流水记录
$this->inventory->insert($inv_data, false);

// 2. 后更新 item_quantities 表
$item_quantity_data = [
    'item_id'     => $item_id,
    'location_id' => $location_id,
    'quantity'    => $item_quantity->quantity + parse_quantity($this->request->getPost('newquantity'))
];
$this->item_quantity->save_value($item_quantity_data, $item_id, $location_id);
```

**特点**:
- 先插入 `inventory`，后更新 `item_quantities`（与 postSave 顺序相反）
- `trans_inventory` 直接存储用户输入的 **增减量**
- 无数据库事务包裹
- 操作顺序与 2.1 相反，增加了不一致风险

### 2.3 销售出库

**模型**: [Sale.php::save_value()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Models/Sale.php#L518-L688)

```php
// 代码片段 (Lines 635-666)
if ($cur_item_info->stock_type == HAS_STOCK && $sale_status == COMPLETED) {
    // 1. 更新 item_quantities 表
    $item_quantity_data = $item_quantity->get_item_quantity($item_data['item_id'], $item_data['item_location']);
    $item_quantity->save_value([
        'quantity'    => $item_quantity_data->quantity - $item_data['quantity'],
        'item_id'     => $item_data['item_id'],
        'location_id' => $item_data['item_location']
    ], $item_data['item_id'], $item_data['item_location']);
    
    // 2. 插入 inventory 流水记录
    $inv_data = [
        'trans_date'      => date('Y-m-d H:i:s'),
        'trans_items'     => $item_data['item_id'],
        'trans_user'      => $employee_id,
        'trans_location'  => $item_data['item_location'],
        'trans_comment'   => 'POS ' . $sale_id,
        'trans_inventory' => -$item_data['quantity']  // 负数表示出库
    ];
    $inventory->insert($inv_data, false);
}
```

**特点**:
- 先更新 `item_quantities`，后插入 `inventory`
- 在数据库事务内执行 (`$this->db->transStart()` / `transComplete()`)
- `trans_inventory` 为负值表示出库
- 仅当 `stock_type == HAS_STOCK` 且 `sale_status == COMPLETED` 时才更新库存

### 2.4 收货入库

**模型**: [Receiving.php::save_value()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Models/Receiving.php#L103-L190)

```php
// 代码片段 (Lines 161-183)
// 1. 更新 item_quantities 表
$item_quantity_value = $item_quantity->get_item_quantity($item_data['item_id'], $item_data['item_location']);
$item_quantity->save_value([
    'quantity'    => $item_quantity_value->quantity + $items_received,
    'item_id'     => $item_data['item_id'],
    'location_id' => $item_data['item_location']
], $item_data['item_id'], $item_data['item_location']);

// 2. 插入 inventory 流水记录
$inv_data = [
    'trans_date'      => date('Y-m-d H:i:s'),
    'trans_items'     => $item_data['item_id'],
    'trans_user'      => $employee_id,
    'trans_location'  => $item_data['item_location'],
    'trans_comment'   => 'RECV ' . $receiving_id,
    'trans_inventory' => $items_received  // 正数表示入库
];
$inventory->insert($inv_data, false);
```

**特点**:
- 先更新 `item_quantities`，后插入 `inventory`
- 在数据库事务内执行
- `trans_inventory` 为正值表示入库

### 2.5 删除销售/收货时的库存回滚

**删除销售**: [Sale.php::delete()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Models/Sale.php#L795-L839)

**删除收货**: [Receiving.php::delete_value()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Models/Receiving.php#L218-L260)

```php
// 删除销售时的库存回滚 (Sale.php Lines 814-829)
if ($update_inventory && $sale_status == COMPLETED) {
    foreach ($items as $item_data) {
        if ($cur_item_info->stock_type == HAS_STOCK) {
            // 1. 插入 inventory 流水记录（反向操作）
            $inv_data = [
                'trans_inventory' => $item_data['quantity_purchased'],  // 正值（加回库存）
                'trans_comment'   => 'Deleting sale ' . $sale_id,
                // ...
            ];
            $inventory->insert($inv_data, false);
            
            // 2. 更新 item_quantities 表
            $item_quantity->change_quantity($item_data['item_id'], $item_data['item_location'], $item_data['quantity_purchased']);
        }
    }
}
```

**特点**:
- 先插入 `inventory`，后更新 `item_quantities`（顺序与正常操作相反）
- 在数据库事务内执行

## 3. 库存流水的记录机制

### 3.1 流水记录的构成

每次库存变化都会在 `inventory` 表中产生一条记录，核心字段：

| 操作类型 | trans_comment | trans_inventory |
|----------|---------------|-----------------|
| 手动编辑商品数量 | `Items.manually_editing_of_quantity` | 差值 (新-旧) |
| 库存调整表单 | 用户输入的备注 | 用户输入的增减量 |
| 销售出库 | `POS {sale_id}` | -销售数量 |
| 收货入库 | `RECV {receiving_id}` | +收货数量 |
| 删除销售 | `Deleting sale {sale_id}` | +销售数量 |
| 删除收货 | `Deleting receiving {receiving_id}` | -收货数量 |
| 删除商品 | `Items.is_deleted` | -当前库存总和 |

### 3.2 库存汇总查询

**模型方法**: [Inventory.php::get_inventory_sum()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Models/Inventory.php#L91-L99)

```php
public function get_inventory_sum(int $item_id): array
{
    $builder = $this->db->table('inventory');
    $builder->select('SUM(trans_inventory) AS sum, MAX(trans_location) AS location_id');
    $builder->where('trans_items', $item_id);
    $builder->groupBy('trans_location');
    return $builder->get()->getResultArray();
}
```

**理论上**，对于任一商品在任一仓库：
```
item_quantities.quantity = SUM(inventory.trans_inventory)
```

## 4. 负库存边界检查

### 4.1 库存检查逻辑

**位置**: [Sale_lib.php::out_of_stock()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Libraries/Sale_lib.php#L1193-L1212)

```php
public function out_of_stock(int $item_id, int $item_location): string
{
    if ($item_id != -1) {
        $item_info = $this->item->get_info_by_id_or_number($item_id);
        
        if ($item_info->stock_type == HAS_STOCK) {
            $item_quantity = $this->item_quantity->get_item_quantity($item_id, $item_location)->quantity;
            $quantity_added = $this->get_quantity_already_added($item_id, $item_location);
            
            if ($item_quantity - $quantity_added < 0) {
                return lang('Sales.quantity_less_than_zero');  // 库存不足
            } elseif ($item_quantity - $quantity_added < $item_info->reorder_level) {
                return lang('Sales.quantity_less_than_reorder_level');  // 低于预警线
            }
        }
    }
    return '';
}
```

### 4.2 检查时机

| 场景 | 检查位置 | 是否阻止操作 |
|------|----------|------------|
| 添加商品到购物车 | [Sale_lib.php::add_item()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Libraries/Sale_lib.php#L1346) | 仅警告，不阻止 |
| 添加套装商品到购物车 | [Sale_lib.php::add_item_kit()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Libraries/Sale_lib.php#L1346) | 仅警告，不阻止 |
| 完成销售时 | 无检查 | 直接扣减 |

### 4.3 负库存边界问题

**问题 1**: 检查仅在前端添加商品时进行，**实际扣减库存时无二次检查**

```php
// Sale.php::save_value() - 实际扣减时没有检查库存是否为负
$item_quantity->save_value([
    'quantity' => $item_quantity_data->quantity - $item_data['quantity'],  // 可能变成负数
    // ...
]);
```

**问题 2**: 并发场景下的检查失效

购物车检查时的库存数量，与实际扣减时的库存数量可能不一致（其他销售已扣减）。

**问题 3**: 手动调整和收货/删除操作无负库存检查

- `postSave()` 和 `postSaveInventory()` 可以将库存设为任意值（包括负数）
- 删除销售时会直接加回库存，无检查

## 5. 一致性问题与风险

### 5.1 数据不一致的风险点

| 风险点 | 位置 | 说明 |
|--------|------|------|
| 操作顺序不一致 | `postSave()` vs `postSaveInventory()` | 前者先更数量后插流水，后者相反 |
| 缺少事务包裹 | `postSave()`, `postSaveInventory()` | 两个操作不在事务中，可能部分成功部分失败 |
| 删除商品时的重置 | [Item.php::delete()](file:///d:/fz/0601-1/solo-dogfeeding/code/12-opensourcepos/app/Models/Item.php#L484-L504) | 先重置 `item_quantities`，后插入 `inventory` 流水 |

### 5.2 删除商品时的库存重置

```php
// Item.php::delete() (Lines 486-499)
$this->db->transStart();

// 1. 重置 item_quantities 为 0
$item_quantity = model(Item_quantity::class);
$item_quantity->reset_quantity($item_id);

// 2. 标记商品为已删除
$builder->where('item_id', $item_id);
$success = $builder->update(['deleted' => 1]);

// 3. 插入 inventory 流水（冲减当前库存）
$inventory = model(Inventory::class);
$success &= $inventory->reset_quantity($item_id);

$this->db->transComplete();
```

### 5.3 理论校验公式

为验证一致性，可执行以下 SQL 查询：

```sql
-- 查询库存数量与流水汇总不一致的记录
SELECT 
    iq.item_id,
    iq.location_id,
    iq.quantity AS current_quantity,
    COALESCE(SUM(inv.trans_inventory), 0) AS inventory_sum,
    iq.quantity - COALESCE(SUM(inv.trans_inventory), 0) AS diff
FROM item_quantities iq
LEFT JOIN inventory inv ON inv.trans_items = iq.item_id AND inv.trans_location = iq.location_id
GROUP BY iq.item_id, iq.location_id
HAVING diff != 0;
```

## 6. 改进建议

### 6.1 代码层面改进

1. **统一操作顺序**: 所有库存调整都采用相同的顺序（建议先插流水后更数量，或相反）
2. **添加事务包裹**: `postSave()` 和 `postSaveInventory()` 应使用数据库事务
3. **增加负库存二次检查**: 在 `Sale.php::save_value()` 实际扣减前再次检查库存
4. **使用乐观锁或行锁**: 并发场景下防止超卖

### 6.2 数据校验

定期运行一致性校验脚本，及时发现并修复不一致数据。

### 6.3 业务规则明确

- 明确是否允许负库存（目前代码逻辑上允许，但前端有警告）
- 明确负库存的处理策略（阻止、警告、允许）
