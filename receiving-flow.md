# 采购收货流程代码分析

## 一、整体架构概览

采购收货流程涉及三层架构：

| 层级       | 核心文件                                    | 主要职责                          |
|------------|--------------------------------------------|-----------------------------------|
| 控制器层   | [Receivings.php](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Controllers/Receivings.php) | HTTP请求处理、参数校验、页面渲染  |
| 业务逻辑层 | [Receiving_lib.php](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Libraries/Receiving_lib.php) | 购物车管理、临时数据存储（Session）|
| 数据模型层 | [Receiving.php](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Models/Receiving.php) | 数据库事务、库存更新、成本计算    |

关联模型：
- [Item.php](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Models/Item.php) - 商品信息、成本价格计算
- [Item_quantity.php](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Models/Item_quantity.php) - 库存数量维护
- [Inventory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Models/Inventory.php) - 库存变动历史记录
- [Supplier.php](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Models/Supplier.php) - 供应商信息

---

## 二、收货保存流程详解

### 2.1 入口控制器：postComplete()

**位置**：[Receivings.php#L325-L379](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Controllers/Receivings.php#L325-L379)

```php
public function postComplete(): string
{
    // 1. 收集数据
    $data['cart'] = $this->receiving_lib->get_cart();
    $data['total'] = $this->receiving_lib->get_total();
    $data['payment_type'] = $this->request->getPost('payment_type');
    // ... 其他数据收集

    // 2. 核心保存调用
    $data['receiving_id'] = 'RECV ' . $this->receiving->save_value(
        $data['cart'],
        $supplier_id,
        $employee_id,
        $data['comment'],
        $data['reference'],
        $data['payment_type'],
        $data['stock_location']
    );

    // 3. 清空购物车
    $this->receiving_lib->clear_all();
}
```

### 2.2 核心保存逻辑：save_value()

**位置**：[Receiving.php#L103-L190](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Models/Receiving.php#L103-L190)

这是一个**数据库事务**，所有操作要么全部成功，要么全部回滚。

#### 事务步骤：

```
┌─────────────────────────────────────────────────────────┐
│                     事务开始 (transStart)               │
├─────────────────────────────────────────────────────────┤
│  1. 插入 receivings 表（收货主记录）                    │
│     - receiving_time, supplier_id, employee_id         │
│     - payment_type, comment, reference                 │
│     → 返回 receiving_id                                │
├─────────────────────────────────────────────────────────┤
│  2. 遍历购物车中的每个商品：                            │
│                                                         │
│     2.1 插入 receivings_items 表（收货明细）            │
│          - receiving_id, item_id, line                 │
│          - quantity_purchased, receiving_quantity      │
│          - item_cost_price, item_unit_price            │
│          - discount, discount_type, item_location      │
│                                                         │
│     2.2 计算实际收货数量：                              │
│          items_received = quantity * receiving_quantity │
│          (receiving_quantity 为包装规格，如一箱=12个)   │
│                                                         │
│     2.3 更新成本价格（如果配置开启）                    │
│          change_cost_price()                           │
│                                                         │
│     2.4 更新库存数量                                    │
│          item_quantity->save_value()                   │
│                                                         │
│     2.5 插入库存变动历史记录                            │
│          inventory->insert()                           │
│                                                         │
│     2.6 复制商品属性链接                                │
│          attribute->copy_attribute_links()             │
├─────────────────────────────────────────────────────────┤
│                     事务提交 (transComplete)            │
└─────────────────────────────────────────────────────────┘
```

**关键代码片段**：
```php
// 行 L124-L187
$this->db->transStart();

// 插入收货主记录
$builder->insert($receivings_data);
$receiving_id = $this->db->insertID();

// 遍历每个商品
foreach ($items as $line => $item_data) {
    // 插入收货明细
    $builder->insert($receivings_items_data);
    
    $items_received = $item_data['receiving_quantity'] != 0 
        ? $item_data['quantity'] * $item_data['receiving_quantity'] 
        : $item_data['quantity'];
    
    // 更新成本价格
    if ($cur_item_info->cost_price != $item_data['price'] 
        && $config['receiving_calculate_average_price']) {
        $item->change_cost_price($item_data['item_id'], $items_received, 
            $item_data['price'], $cur_item_info->cost_price);
    }
    
    // 更新库存数量
    $item_quantity->save_value([...]);
    
    // 插入库存变动记录
    $inventory->insert($inv_data, false);
}

$this->db->transComplete();
```

---

## 三、库存更新逻辑

### 3.1 实际收货数量计算

**位置**：[Receiving.php#L154](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Models/Receiving.php#L154)

```php
$items_received = $item_data['receiving_quantity'] != 0 
    ? $item_data['quantity'] * $item_data['receiving_quantity'] 
    : $item_data['quantity'];
```

- `receiving_quantity`：包装规格（如 1 箱 = 12 个，则此值为 12）
- `quantity`：采购的包装数量
- 如果 `receiving_quantity` 为 0 或 1，则直接使用 `quantity`

> **注意三元判断**：`rq != 0` 为真时走乘法，为假时直接用 `quantity`。这是一个**兜底逻辑**，会在删除时造成问题。

### 3.2 库存数量更新

**位置**：[Item_quantity.php#L45-L57](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Models/Item_quantity.php#L45-L57)

`save_value()` 方法：
- 若该商品在该仓库不存在记录，则 INSERT
- 若已存在，则 UPDATE

```php
// Receiving.php 中调用
$item_quantity_value = $item_quantity->get_item_quantity($item_data['item_id'], $item_data['item_location']);
$item_quantity->save_value(
    [
        'quantity'    => $item_quantity_value->quantity + $items_received,
        'item_id'     => $item_data['item_id'],
        'location_id' => $item_data['item_location']
    ],
    $item_data['item_id'],
    $item_data['item_location']
);
```

### 3.3 库存变动历史记录

**位置**：[Inventory.php#L14-L27](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Models/Inventory.php#L14-L27)

每次库存变动都会在 `inventory` 表中留下记录：

```php
$inv_data = [
    'trans_date'      => date('Y-m-d H:i:s'),
    'trans_items'     => $item_data['item_id'],
    'trans_user'      => $employee_id,
    'trans_location'  => $item_data['item_location'],
    'trans_comment'   => 'RECV ' . $receiving_id,
    'trans_inventory' => $items_received
];
$inventory->insert($inv_data, false);
```

`trans_comment` 字段格式为 `RECV {receiving_id}`，用于追溯来源。

### 3.4 receiving_quantity 的来源与数据流

`receiving_quantity`（包装规格）的取值来源有三层：

```
1. 商品默认值（items 表）
   ↓ [Receiving_lib.php#L317-L319]
2. 加入购物车时默认使用商品表值，可被显式覆盖
   ↓ [Receiving_lib.php#L342]
3. 存入 Session 购物车 → 保存时写入 receivings_items 表
   ↓ [Receiving.php#L144]
4. 删除时从 receivings_items 表读出并用于回滚
```

**来源 1：商品默认值**
```php
// Receiving_lib.php#L308-L319
if ($itemInfo->receiving_quantity == 0 || $itemInfo->receiving_quantity == 1) {
    $receivingQuantityChoices = [1 => 'x1'];     // 只提供 x1 选项
} else {
    $receivingQuantityChoices = [
        to_quantity_decimals($itemInfo->receiving_quantity) => 'x' . $itemInfo->receiving_quantity,
        1 => 'x1'
    ];
}

if (is_null($receivingQuantity)) {
    $receivingQuantity = $itemInfo->receiving_quantity;  // 用商品默认值
}
```

**来源 2：用户编辑覆盖**
```php
// Receiving_lib.php#L371-L392
public function edit_item($line, ..., float $receiving_quantity): bool
{
    $line['receiving_quantity'] = $receiving_quantity;  // 用户可修改为任意值（包括 0~1 之间）
}
```

因此 `receiving_quantity` 可能的值包括：
- 商品表定义的包装规格（如 6、12、24 等）
- 1（按个采购）
- 0（商品未设置包装规格的默认值）
- **0 < rq < 1 的小数**（用户手动编辑，例如 0.5 表示半箱，理论上不推荐但代码允许）

---

## 四、成本计算方式：移动加权平均法

### 4.1 触发条件

**位置**：[Receiving.php#L157-L159](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Models/Receiving.php#L157-L159)

```php
if ($cur_item_info->cost_price != $item_data['price'] 
    && $config['receiving_calculate_average_price']) {
    $item->change_cost_price($item_data['item_id'], $items_received, 
        $item_data['price'], $cur_item_info->cost_price);
}
```

触发条件：
1. 本次采购价格 ≠ 当前成本价格
2. 配置项 `receiving_calculate_average_price` = true

### 4.2 平均成本计算公式

**位置**：[Item.php#L1068-L1088](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Models/Item.php#L1068-L1088)

```php
public function change_cost_price(int $item_id, float $items_received, float $new_price, ?float $old_price = null): bool
{
    // 1. 查询所有仓库的当前库存总量
    $builder->selectSum('quantity');
    $builder->where('item_id', $item_id);
    $builder->join('stock_locations', 'stock_locations.location_id=item_quantities.location_id');
    $builder->where('stock_locations.deleted', 0);
    $old_total_quantity = $builder->get()->getRow()->quantity;
    
    // 2. 计算收货后的总库存
    $total_quantity = $old_total_quantity + $items_received;
    
    // 3. 移动加权平均公式
    // 平均成本 = (旧库存总值 + 新收货总值) / 总库存数量
    $average_price = bcdiv(
        bcadd(
            bcmul((string)$items_received, (string)$new_price),      // 新收货总值
            bcmul((string)$old_total_quantity, (string)$old_price)   // 旧库存总值
        ),
        (string)$total_quantity
    );
    
    // 4. 更新商品成本价格
    $data = ['cost_price' => $average_price];
    return $this->save_value($data, $item_id);
}
```

**公式详解**：
```
平均成本 = (旧库存数量 × 旧成本 + 新收货数量 × 新价格) / (旧库存数量 + 新收货数量)
```

> **重要**：这里使用的是**所有仓库的库存总量**来计算平均成本，而不是仅使用收货仓库的数量。

### 4.3 ⚠️ 关键执行顺序：先算成本价，再改库存

**代码证据**：[Item.php#L1065](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Models/Item.php#L1065) 的注释明确警告：

```php
// caution: must be used before item_quantities gets updated, otherwise the average price is wrong!
```

在 [Receiving.php#L156-L171](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Models/Receiving.php#L156-L171) 中实际执行顺序为：

```
① change_cost_price()   ← 先算成本价
   │
   └─ 查询 item_quantities 表获取 旧库存数量（收货前）
   └─ 用旧库存计算加权平均
   └─ 只更新 items.cost_price，不改 item_quantities
        │
        ▼
② item_quantity->save_value()  ← 后改库存数量
   │
   └─ 更新 item_quantities.quantity = 旧库存 + 收货数量
```

#### 为什么必须是这个顺序？

`change_cost_price()` 内部需要查询 **收货前的旧库存**：

```php
// Item.php#L1075-L1080
$builder->selectSum('quantity');
$builder->from('item_quantities');
// ...
$old_total_quantity = $builder->get()->getRow()->quantity;  // 查询的是当前 DB 中的值
```

#### 如果反过来（先改库存，再算成本）会发生什么？

假设场景：
- 旧库存 = 100 个，旧成本 = 10 元
- 新收货 = 50 个，新价格 = 14 元

**正确顺序（先算成本 → 再改库存）**：
```
① change_cost_price():
   读取 item_quantities → 旧库存 = 100
   平均成本 = (100×10 + 50×14) / (100+50) = 1700/150 = 11.33 元
   UPDATE items SET cost_price = 11.33

② item_quantity->save_value():
   UPDATE item_quantities SET quantity = 100 + 50 = 150

最终结果：库存 150，成本 11.33 元 ✅ 正确
```

**错误顺序（先改库存 → 再算成本）**：
```
① item_quantity->save_value():
   UPDATE item_quantities SET quantity = 100 + 50 = 150

② change_cost_price():
   读取 item_quantities → 旧库存 = 150  ← 已经是收货后的值！
   平均成本 = (150×10 + 50×14) / (150+50) = 2200/200 = 11.00 元  ← 偏低
   总库存又加了一次 50 → 最终库存 = 200  ← 多加了一次！

最终结果：库存 200（虚增50），成本 11.00 元 ❌ 双重错误
```

> **设计本质**：`change_cost_price()` 依赖的是"收货前"的数据库快照，而不是传入参数。这是一个**隐式副作用依赖**，必须严格保证调用时序。

---

## 五、取消收货（删除）的回滚机制

删除收货单时，库存是否会被反向扣回，取决于**两层逻辑**，层层递进：

```
┌───────────────────────────────────────────────────────────┐
│  第一层：update_inventory 开关（总闸门）                    │
│  - 位置：删除入口参数                                     │
│  - 决定"要不要扣"                                          │
│  - 开关 = false → 完全不碰库存，只删单据                    │
│  - 开关 = true  → 进入第二层判断                           │
└──────────────────────┬────────────────────────────────────┘
                       ▼
┌───────────────────────────────────────────────────────────┐
│  第二层：receiving_quantity 计算逻辑（扣多少）              │
│  - 位置：delete_value() 内部循环                           │
│  - 决定"扣多少"                                            │
│  - rq ≠ 0 → 对称扣回（正确）                               │
│  - rq = 0 → 不对称，只加不扣（缺陷）                        │
└───────────────────────────────────────────────────────────┘
```

### 5.1 第一层关系：update_inventory 库存同步开关

#### 5.1.1 开关的传递路径

**位置**：从控制器 [Receivings.php#L288](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Controllers/Receivings.php#L288) 一直传到模型底层。

```
postDelete($update_inventory = true)        [控制器入口]
    ↓ 传递
delete_list($receiving_ids, $update_inventory)   [批量删除]
    ↓ 逐个传递
delete_value($receiving_id, $update_inventory)   [单个删除]
    ↓ if 判断
IF update_inventory = true → 执行库存回滚
ELSE → 跳过库存，仅删除单据
```

#### 5.1.2 控制器入口定义

```php
// Receivings.php#L288
public function postDelete(int $receiving_id = -1, bool $update_inventory = true): ResponseInterface
```

- 参数默认值为 `true`，即默认同步库存
- 这是**总闸门**：关闭则完全不碰库存表（`item_quantities` 和 `inventory`）

#### 5.1.3 delete_value() 中的开关判断

**位置**：[Receiving.php#L223](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Models/Receiving.php#L223)

```php
public function delete_value(int $receiving_id, int $employee_id, bool $update_inventory = true): bool
{
    $this->db->transStart();

    if ($update_inventory) {
        // → 只有开关开启，才会进入库存回滚逻辑
        $items = $this->get_receiving_items($receiving_id)->getResultArray();
        foreach ($items as $item) {
            // 插入反向库存变动记录
            $inventory->insert($inv_data, false);
            // 反向更新库存数量
            $item_quantity->change_quantity(...);
        }
    }

    // 无论开关是否开启，都会删除单据本身
    $builder->delete(['receiving_id' => $receiving_id]);  // 删除明细
    $builder->delete(['receiving_id' => $receiving_id]);  // 删除主记录

    $this->db->transComplete();
    return $this->db->transStatus();
}
```

#### 5.1.4 开关两种状态的对比

| `update_inventory` | 收货单据 | item_quantities（库存数量） | inventory（变动历史） | 成本价格 |
|---|---|---|---|---|
| **true**（默认） | 删除 | 修改（反向扣回） | 插入（反向记录） | 不回滚 |
| **false** | 删除 | **不变** | **不变** | 不回滚 |

> **注意**：无论开关如何，成本价格都不会回滚（这是另一个独立缺陷，见 5.3 节）。

---

### 5.2 第二层关系：receiving_quantity 计算不对称问题

> 本层讨论的是**在 `update_inventory = true` 的前提下**，库存扣回数量是否准确的问题。

#### 5.2.1 保存 vs 删除计算公式对比

| 阶段 | 代码位置 | 计算公式 | 是否有兜底 |
|---|---|---|---|
| **保存时** | [Receiving.php#L154](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Models/Receiving.php#L154) | `rq != 0 ? qty*rq : qty` | **有**（rq=0 时兜底用 qty） |
| **删除时** | [Receiving.php#L238/L244](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Models/Receiving.php#L238-L244) | `qty * (-rq)` | **无**（直接乘） |

**保存时（含兜底三元运算）**：
```php
// L154
$items_received = $item_data['receiving_quantity'] != 0 
    ? $item_data['quantity'] * $item_data['receiving_quantity'] 
    : $item_data['quantity'];
```

**删除时（直接乘法，无兜底）**：
```php
// L238 库存变动记录
'trans_inventory' => $item['quantity_purchased'] * (-$item['receiving_quantity'])
// L244 库存数量
$item_quantity->change_quantity(..., 
    $item['quantity_purchased'] * (-$item['receiving_quantity'])
);
```

#### 5.2.2 不同取值范围的行为矩阵

以 `quantity_purchased = 10` 为例，覆盖全部情况：

| `receiving_quantity` | 保存时库存变化 | 删除时库存变化 | 净变化 | 对称吗？ | 所属分支 |
|---|---|---|---|---|---|
| **rq = 0** | `rq≠0` 为假 → 兜底 **+10** | `10 × (-0) = 0` | **+10** ❌ | 🔴 不对称 | 三元分支 else |
| **0 < rq < 1**（如 0.5） | `10 × 0.5 = +5` | `10 × (-0.5) = -5` | **0** ✅ | 对称 | 三元分支 if |
| **rq = 1** | `rq≠0` 为真 → `10×1 = +10` | `10 × (-1) = -10` | **0** ✅ | 对称 | 三元分支 if |
| **rq > 1**（如 12） | `10 × 12 = +120` | `10 × (-12) = -120` | **0** ✅ | 对称 | 三元分支 if |

#### 5.2.3 执行路径差异可视化

```
保存时（save_value）:
    rq = 0?
    ├─ 是 → items_received = quantity          ← 兜底分支（else）
    └─ 否 → items_received = quantity * rq     ← 乘法分支（if）

删除时（delete_value）:
    无论 rq 是多少
    └─ 一律 → change = quantity * (-rq)        ← 始终乘法，无分支
```

**不对称的根源**：保存时在 rq=0 处有一个"跳变"（从乘法跳到直接用 quantity），删除时没有对应的跳变，始终走乘法。

#### 5.2.4 rq=0 场景的完整数据流

```
保存方向：
  add_item() [Receiving_lib]
    └─ 商品默认 rq = 0（未设置包装规格）
    └─ 存入 Session
         └─ save_value() [Receiving]
              └─ rq ≠ 0 ? qty*rq : qty
                   └─ rq=0 → 走 else → 库存 +qty
                   └─ receivings_items 表保存 rq = 0

删除方向：
  delete_value() [Receiving]
    └─ get_receiving_items() → 读出 rq = 0
         └─ change = qty * (-0) = 0
         └─ 库存数量不变，库存变动记录写入 0
```

**结论**：rq=0 的商品，收货时库存增加了 `quantity`，删除时只减了 `0`，差额永久留在库存中。

---

### 5.3 成本价格的回滚缺失

**位置**：[Receiving.php#L218-L260](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Models/Receiving.php#L218-L260)

在 `delete_value()` 方法中，**没有调用 `change_cost_price()` 来回滚成本价格**。

这意味着无论 `update_inventory` 开关是否开启：
- ❌ 成本价格：保持收货后的平均价格，**不会自动恢复到收货前的成本**

> 代码中的 TODO 注释也提到了这个问题（第224行）：
> ```
> // TODO: defect, not all item deletions will be undone?
> ```

---

### 5.4 删除回滚完整流程图

```
postDelete(receiving_id, update_inventory=true)
    ↓
delete_list(receiving_ids, employee_id, update_inventory)
    ↓ 事务开始
    ↓ 逐个调用 delete_value()
    ↓
    ┌─ IF update_inventory = true ────────────────────────┐
    │   遍历收货明细：                                       │
    │   ├─ 插入 inventory 反向记录（值 = qty*(-rq)）       │
    │   │   └─ rq=0 时 = 0（不对称缺陷）                     │
    │   └─ item_quantity->change_quantity(qty*(-rq))      │
    │       └─ rq=0 时 = 0（不对称缺陷）                     │
    └───────────────────────────────────────────────────────┘
    ↓
    删除 receivings_items 明细
    ↓
    删除 receivings 主记录
    ↓
    事务提交
```

---
```php
// L154
$items_received = $item_data['receiving_quantity'] != 0 
    ? $item_data['quantity'] * $item_data['receiving_quantity'] 
    : $item_data['quantity'];   // ← 当 rq=0 时，兜底用 quantity
```

**删除时（无兜底）**：
```php
// L238 库存变动记录
'trans_inventory' => $item['quantity_purchased'] * (-$item['receiving_quantity'])
// L244 库存数量
$item_quantity->change_quantity(..., 
    $item['quantity_purchased'] * (-$item['receiving_quantity'])
);
// ← 当 rq=0 时，直接乘 0，结果为 0！
```

#### 5.5.2 receiving_quantity 不同取值范围的行为分析

以 `quantity_purchased = 10` 为例，覆盖所有情况：

| `receiving_quantity` | 保存时库存变化 | 删除时库存变化 | 净变化 | 是否一致？ |
|---|---|---|---|---|
| **rq = 0** | `10 * 0 ≠ 0` 为假 → 取兜底 **+10** | `10 × (-0) = 0` | **+10** ❌ | 🔴 **严重不一致** |
| **0 < rq < 1** (如 0.5) | `10 × 0.5 = +5` | `10 × (-0.5) = -5` | **0** ✅ | 一致 |
| **rq = 1** | `1 ≠ 0` 为真 → `10 × 1 = +10` | `10 × (-1) = -10` | **0** ✅ | 一致 |
| **rq > 1** (如 12) | `10 × 12 = +120` | `10 × (-12) = -120` | **0** ✅ | 一致 |

#### 5.5.3 用户问题的精确答案

**问题一：删除收货单时，库存是否必然反向扣回？**

**答案：不必然。**

只有当 `receiving_quantity ≠ 0` 时，库存才会对称反向扣回。当 `receiving_quantity = 0` 时，删除操作**完全不扣回**，导致库存永久虚增。

具体场景：
- 商品未设置包装规格（`items.receiving_quantity = 0`）
- 收货 10 个，保存时库存 +10（兜底用了 quantity）
- 删除该收货单，库存变动 = `10 × (-0) = 0`
- 结果：这 10 个库存**永久留在库存中无法消除**

**问题二：为什么收货倍数在 0 到 1 之间时前后处理会有差异？**

**答案：严格来说，差异仅发生在 rq=0 的边界点，而非 0 < rq < 1 区间。**

- **0 < rq < 1 区间**：因为 `rq != 0` 判断为真，保存时走乘法分支，与删除时对称，**行为一致**。
  - 例：rq=0.5, qty=10 → 保存时 +5，删除时 -5 ✅

- **rq = 0 的边界点**：保存时走兜底分支用 qty，删除时直接乘 0，**行为严重不一致**。
  - 例：rq=0, qty=10 → 保存时 +10（兜底），删除时 -0=0 ❌

> **执行路径清晰化**：
> 保存时的三元运算符 `rq != 0 ? qty×rq : qty` 在 rq=0 时执行路径发生"跳变"，而删除时始终走乘法，这就是前后差异的来源。

#### 5.5.4 数据保存的完整路径追溯

```
保存时数据流：
  add_item() [Receiving_lib]
    └─ 若 receivingQuantity = null → 取商品默认值（可能为 0）
    └─ 存入 Session recv_cart
         └─ save_value() [Receiving]
              └─ rq != 0 ? qty*rq : qty   ← 此处兜底生效
                   └─ 写入 item_quantities (+qty 或 +qty*rq)
                   └─ 写入 inventory 表 (+qty 或 +qty*rq)
              └─ receivings_items 表中 rq 原样保存为 0

删除时数据流：
  delete_value() [Receiving]
    └─ get_receiving_items() → 读出 rq = 0
         └─ change_quantity(qty × (-0) = 0)  ← 无兜底，扣回 0
         └─ inventory 表写入 0
```

**根本原因**：保存时的三元判断逻辑**只做了单向兜底**（rq=0 时兜底），删除时没有对应的反向兜底逻辑。

---

## 六、供应商关联逻辑

### 6.1 关联方式

**位置**：[Receiving.php#L117](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Models/Receiving.php#L117)

```php
$receivings_data = [
    // ...
    'supplier_id' => $supplier->exists($supplier_id) ? $supplier_id : null,
    // ...
];
```

- 收货单通过 `supplier_id` 字段关联供应商
- 关联前会校验供应商是否存在，不存在则设为 `null`
- 保存在 `receivings` 表的主记录中

### 6.2 对供应商账户的影响

**重要发现**：**采购收货不直接更新供应商账户余额**。

- 系统中没有应付账款（Accounts Payable）的自动记账功能
- `suppliers` 表中也没有余额字段
- 收货单仅记录供应商关联，用于统计和追溯
- 付款方式（`payment_type`）只是记录字段，不会触发任何财务记账

供应商模型 [Supplier.php](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Models/Supplier.php) 中仅包含基本信息：
- `company_name` - 公司名称
- `account_number` - 账号
- `tax_id` - 税号
- `category` - 类别（货物供应商/费用供应商）

**没有**应付账款余额、信用额度等财务字段。

---

## 七、三种收货模式

**位置**：[Receiving_lib.php#L103-L119](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Libraries/Receiving_lib.php#L103-L119)

系统支持三种收货模式，存储在 Session 的 `recv_mode` 中：

| 模式          | 说明               | 库存影响                     |
|---------------|--------------------|------------------------------|
| `receive`     | 正常采购收货       | 库存数量 + 收货数量          |
| `return`      | 退货               | 库存数量 - 退货数量（数量为负） |
| `requisition` | 仓库调拨           | 源仓库 - 数量，目的仓库 + 数量 |

### 调拨模式特殊处理

**位置**：[Receivings.php#L388-L403](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Controllers/Receivings.php#L388-L403)

```php
public function postRequisitionComplete(): string
{
    // 将每个商品拆分为两条记录：
    // 1. 目的仓库：+数量
    // 2. 源仓库：-数量
    foreach ($this->receiving_lib->get_cart() as $item) {
        $this->receiving_lib->delete_item($item['line']);
        $this->receiving_lib->add_item($item['item_id'], $item['quantity'], 
            $this->receiving_lib->get_stock_destination(), $item['discount_type']);
        $this->receiving_lib->add_item($item['item_id'], -$item['quantity'], 
            $this->receiving_lib->get_stock_source(), $item['discount_type']);
    }
    
    return $this->postComplete();
}
```

---

## 八、Session 购物车机制

**位置**：[Receiving_lib.php#L45-L61](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Libraries/Receiving_lib.php#L45-L61)

收货过程中，所有临时数据存储在 Session 中：

| Session 键名         | 内容                     |
|----------------------|--------------------------|
| `recv_cart`          | 购物车商品列表           |
| `recv_supplier`      | 当前选中的供应商 ID      |
| `recv_mode`          | 收货模式（receive/return/requisition） |
| `recv_stock_source`  | 源仓库 ID                |
| `recv_stock_destination` | 目的仓库 ID          |
| `recv_comment`       | 备注                     |
| `recv_reference`     | 参考单号                 |
| `recv_print_after_sale` | 是否打印收据          |

### 清空操作

**位置**：[Receiving_lib.php#L468-L475](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Libraries/Receiving_lib.php#L468-L475)

```php
public function clear_all(): void
{
    $this->clear_mode();
    $this->empty_cart();
    $this->remove_supplier();
    $this->clear_comment();
    $this->clear_reference();
}
```

---

## 九、完整调用链总结

### 9.1 保存流程

```
postComplete() [Receivings.php]
    ↓
save_value(cart, supplier_id, employee_id, ...) [Receiving.php]
    ├─ INSERT INTO receivings ...
    ├─ 遍历购物车商品：
    │   ├─ INSERT INTO receivings_items ...
    │   ├─ 计算 items_received = rq != 0 ? qty*rq : qty
    │   ├─ change_cost_price() [Item.php] ← 移动加权平均
    │   │   └─ UPDATE items SET cost_price = ?
    │   ├─ save_value() [Item_quantity.php]
    │   │   └─ INSERT/UPDATE item_quantities SET quantity = ?
    │   └─ insert() [Inventory.php]
    │       └─ INSERT INTO inventory ...
    └─ 返回事务结果
```

### 9.2 取消/删除流程

```
postDelete(receiving_id) [Receivings.php]
    ↓
delete_list(receiving_ids) [Receiving.php]
    ↓
delete_value(receiving_id, employee_id, update_inventory) [Receiving.php]
    ├─ get_receiving_items(receiving_id)
    ├─ 遍历收货明细：
    │   ├─ INSERT INTO inventory (反向记录，值 = qty*(-rq)，无兜底)
    │   └─ change_quantity(item_id, location_id, qty*(-rq)) [Item_quantity.php]
    │       └─ UPDATE item_quantities SET quantity = ?
    ├─ DELETE FROM receivings_items WHERE receiving_id = ?
    └─ DELETE FROM receivings WHERE receiving_id = ?
```

---

## 十、关键数据表结构

### 10.1 receivings 表（收货主记录）

| 字段             | 说明                   |
|------------------|------------------------|
| receiving_id     | 主键，自增             |
| receiving_time   | 收货时间               |
| supplier_id      | 供应商 ID（可空）      |
| employee_id      | 操作员 ID              |
| payment_type     | 付款方式               |
| comment          | 备注                   |
| reference        | 参考单号               |

### 10.2 receivings_items 表（收货明细）

| 字段                | 说明                         |
|---------------------|------------------------------|
| receiving_id        | 外键，关联收货单             |
| item_id             | 外键，关联商品               |
| line                | 行号                         |
| description         | 描述                         |
| serialnumber        | 序列号                       |
| quantity_purchased  | 采购数量（包装数）           |
| receiving_quantity  | 包装规格（每包装数量）       |
| discount            | 折扣                         |
| discount_type       | 折扣类型（0=金额，1=百分比） |
| item_cost_price     | 当时的成本价格               |
| item_unit_price     | 采购单价                     |
| item_location       | 仓库 ID                      |

### 10.3 item_quantities 表（库存数量）

| 字段        | 说明         |
|-------------|--------------|
| item_id     | 商品 ID      |
| location_id | 仓库 ID      |
| quantity    | 当前库存数量 |

### 10.4 inventory 表（库存变动历史）

| 字段             | 说明                     |
|------------------|--------------------------|
| trans_id         | 主键，自增               |
| trans_date       | 变动时间                 |
| trans_items      | 商品 ID                  |
| trans_user       | 操作员 ID                |
| trans_location   | 仓库 ID                  |
| trans_comment    | 变动说明（如 "RECV 123"）|
| trans_inventory  | 变动数量（正=增，负=减） |

---

## 十一、已知问题与注意事项

1. **🔴 receiving_quantity 单向兜底缺陷**：保存时 `rq=0` 会兜底用 quantity，但删除时无对应兜底，导致库存**无法扣回**，永久虚增。

2. **成本价格回滚缺陷**：取消收货时，成本价格不会自动回滚。如果收货时更新了成本价格，删除后成本仍保持新的平均价格。

3. **无应付账款管理**：系统没有应付账款模块，收货不会自动生成对供应商的欠款记录。

4. **成本计算范围**：移动平均成本计算使用**所有仓库的总库存**，而非仅收货仓库的库存。

5. **成本-库存调用时序**：必须严格先算成本价（查旧库存快照），再改库存。若顺序颠倒，会导致库存虚增且成本偏低的**双重错误**。

6. **事务完整性**：所有数据库操作都在事务中执行，保证数据一致性。

7. **库存历史追踪**：所有库存变动都留有完整历史记录，可通过 `trans_comment` 字段追溯来源单据。
