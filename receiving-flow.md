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

---

## 五、取消收货（删除）的回滚机制

### 5.1 控制器入口

**位置**：[Receivings.php#L288-L302](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Controllers/Receivings.php#L288-L302)

```php
public function postDelete(int $receiving_id = -1, bool $update_inventory = true): ResponseInterface
{
    $employee_id = $this->employee->get_logged_in_employee_info()->person_id;
    $receiving_ids = $receiving_id == -1 
        ? $this->request->getPost('ids', FILTER_SANITIZE_NUMBER_INT) 
        : [$receiving_id];
    
    if ($this->receiving->delete_list($receiving_ids, $employee_id, $update_inventory)) {
        // 返回成功
    }
}
```

### 5.2 批量删除：delete_list()

**位置**：[Receiving.php#L196-L213](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Models/Receiving.php#L196-L213)

```php
public function delete_list(array $receiving_ids, int $employee_id, bool $update_inventory = true): bool
{
    $this->db->transStart();
    
    foreach ($receiving_ids as $receiving_id) {
        $success &= $this->delete_value($receiving_id, $employee_id, $update_inventory);
    }
    
    $this->db->transComplete();
    return $success && $this->db->transStatus();
}
```

### 5.3 单个删除回滚：delete_value()

**位置**：[Receiving.php#L218-L260](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Models/Receiving.php#L218-L260)

```
┌─────────────────────────────────────────────────────────┐
│                     事务开始 (transStart)               │
├─────────────────────────────────────────────────────────┤
│  IF update_inventory = true:                            │
│                                                         │
│  1. 查询该收货单的所有明细                              │
│     get_receiving_items($receiving_id)                  │
│                                                         │
│  2. 遍历每个商品，执行反向操作：                         │
│                                                         │
│     2.1 插入反向库存变动记录                            │
│          trans_inventory = quantity_purchased *         │
│                           (-receiving_quantity)         │
│          trans_comment = "Deleting receiving {id}"      │
│                                                         │
│     2.2 反向更新库存数量                                │
│          change_quantity(item_id, location_id,          │
│                          quantity * (-receiving_quantity))│
│                                                         │
│  3. 删除 receivings_items 表中的明细记录                 │
│                                                         │
│  4. 删除 receivings 表中的主记录                        │
│                                                         │
│                     事务提交 (transComplete)            │
└─────────────────────────────────────────────────────────┘
```

**关键代码**：
```php
// 回滚库存变动记录
$inv_data = [
    'trans_date'      => date('Y-m-d H:i:s'),
    'trans_items'     => $item['item_id'],
    'trans_user'      => $employee_id,
    'trans_comment'   => 'Deleting receiving ' . $receiving_id,
    'trans_location'  => $item['item_location'],
    'trans_inventory' => $item['quantity_purchased'] * (-$item['receiving_quantity'])
];
$inventory->insert($inv_data, false);

// 回滚库存数量
$item_quantity->change_quantity(
    $item['item_id'], 
    $item['item_location'], 
    $item['quantity_purchased'] * (-$item['receiving_quantity'])
);
```

### 5.4 ⚠️ 重要缺陷：成本价格未回滚

**位置**：[Receiving.php#L218-L260](file:///d:/fz/0601-1/solo-dogfeeding/code/13-opensourcepos/app/Models/Receiving.php#L218-L260)

在 `delete_value()` 方法中，**没有调用 `change_cost_price()` 来回滚成本价格**。

这意味着：
- ✅ 库存数量：正确回滚（减去收货数量）
- ✅ 库存历史：正确记录反向变动
- ❌ 成本价格：保持收货后的平均价格，**不会自动恢复到收货前的成本**

> **代码中的 TODO 注释也提到了这个问题**（第224行）：
> ```
> // TODO: defect, not all item deletions will be undone?
> ```

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
    │   ├─ 计算 items_received = quantity * receiving_quantity
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
    │   ├─ INSERT INTO inventory (反向记录)
    │   └─ change_quantity(item_id, location_id, -quantity) [Item_quantity.php]
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

1. **成本价格回滚缺陷**：取消收货时，成本价格不会自动回滚。如果收货时更新了成本价格，删除后成本仍保持新的平均价格。

2. **无应付账款管理**：系统没有应付账款模块，收货不会自动生成对供应商的欠款记录。

3. **成本计算范围**：移动平均成本计算使用**所有仓库的总库存**，而非仅收货仓库的库存。

4. **事务完整性**：所有数据库操作都在事务中执行，保证数据一致性。

5. **库存历史追踪**：所有库存变动都留有完整历史记录，可通过 `trans_comment` 字段追溯来源单据。
