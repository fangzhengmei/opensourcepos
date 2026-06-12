# CSV 导入处理链路分析

## 一、整体架构概览

系统存在两条独立的 CSV 导入链路：

| 模块 | 控制器 | 模型 | 列名匹配方式 | 事务处理 |
|------|--------|------|-------------|----------|
| 商品导入 | [Items.php](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Items.php) | [Item.php](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Models/Item.php) | 列名硬编码映射 | 整体事务 |
| 客户导入 | [Customers.php](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Customers.php) | [Customer.php](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Models/Customer.php) | 位置索引匹配 | 无事务 |

---

## 二、CSV 文件读取

### 2.1 读取流程

核心函数位于 [importfile_helper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Helpers/importfile_helper.php)：

```php
// L54-L80
function get_csv_file(string $file_name): array
```

**处理步骤：**
1. 检查 UTF-8 BOM 标记，存在则跳过前 3 字节（[L63-L66](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Helpers/importfile_helper.php#L63-L66)）
2. 读取第一行作为表头
3. 使用 `array_combine` 将表头与数据行关联，形成关联数组
4. 跳过空行（`$row !== [null]`）

### 2.2 CSV 模板生成

```php
// L8-L16
function generate_import_items_csv(array $stock_locations, array $attributes): string
```

**固定列头：**
`Id, Barcode, Item Name, Category, Supplier ID, Cost Price, Unit Price, Tax 1 Name, Tax 1 Percent, Tax 2 Name, Tax 2 Percent, Reorder Level, Description, Allow Alt Description, Item has Serial Number, Image, HSN`

**动态列头：**
- 库存位置：`location_{location_name}`（[L22-L31](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Helpers/importfile_helper.php#L22-L31)）
- 自定义属性：`attribute_{attribute_name}`（[L37-L47](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Helpers/importfile_helper.php#L37-L47)）

---

## 三、列名匹配（字段映射）

### 3.1 商品导入 - 硬编码映射

位于 [Items.php](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Items.php) `postImportCsvFile()` 方法（[L1013-L1024](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Items.php#L1013-L1024)）：

| CSV 列名 | 数据库字段 | 说明 |
|---------|-----------|------|
| `Id` | `item_id` | 主键，>0 表示更新 |
| `Item Name` | `name` | 商品名称 |
| `Description` | `description` | 商品描述 |
| `Category` | `category` | 分类 |
| `Cost Price` | `cost_price` | 成本价 |
| `Unit Price` | `unit_price` | 售价 |
| `Reorder Level` | `reorder_level` | 补货预警线 |
| `HSN` | `hsn_code` | HSN 编码 |
| `Image` | `pic_filename` | 图片文件名 |
| `Supplier ID` | `supplier_id` | 供应商 ID |
| `Barcode` | `item_number` | 条码 |
| `Allow Alt Description` | `allow_alt_description` | 是否允许替代描述 |
| `Item has Serial Number` | `is_serialized` | 是否序列号管理 |

**动态字段匹配：**
- **库存位置**：前缀 `location_`，通过 `substr($key, 9)` 提取位置名（[L1136-L1137](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Items.php#L1136-L1137)）
- **自定义属性**：前缀 `attribute_`，通过 `substr($key, 10)` 提取属性名（[L1113-L1117](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Items.php#L1113-L1117)）

### 3.2 客户导入 - 位置索引匹配

位于 [Customers.php](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Customers.php) `postImportCsvFile()` 方法（[L431-L454](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Customers.php#L431-L454)）：

**⚠️ 风险点：** 客户导入**不使用列名匹配**，而是按数组索引硬编码映射，CSV 列顺序不可变动。

| 索引 | 字段 |
|-----|------|
| $data[0] | first_name |
| $data[1] | last_name |
| $data[2] | gender |
| $data[3] | consent |
| $data[4] | email |
| $data[5] | phone_number |
| $data[6] | address_1 |
| $data[7] | address_2 |
| $data[8] | city |
| $data[9] | state |
| $data[10] | zip |
| $data[11] | country |
| $data[12] | comments |
| $data[13] | company_name |
| $data[14] | account_number |
| $data[15] | discount |
| $data[16] | discount_type |
| $data[17] | taxable |

---

## 四、默认值处理

### 4.1 商品导入默认值

**新增与更新的差异化处理**（[L1030-L1036](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Items.php#L1030-L1036)）：

```php
if ($isUpdate) {
    // 更新：空值保留 null，不做修改
    $itemData['allow_alt_description'] = $row['...'] === '' ? null : $row['...'];
    $itemData['is_serialized'] = $row['...'] === '' ? null : $row['...'];
} else {
    // 新增：空值默认设为 '0'
    $itemData['allow_alt_description'] = $row['...'] === '' ? '0' : '1';
    $itemData['is_serialized'] = $row['...'] === '' ? '0' : '1';
}
```

**其他默认值：**
- `cost_price`：新增时空值默认 `0`（[L1177](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Items.php#L1177)）
- `deleted`：固定 `false`（[L1021](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Items.php#L1021)）
- 库存数量：新增时空值默认 `0`（[L1289-L1294](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Items.php#L1289-L1294)）
- `supplier_id`：不存在时设为 `null`（[L1026-L1028](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Items.php#L1026-L1028)）

**⚠️ 空值过滤陷阱**（[L1053-L1055](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Items.php#L1053-L1055)）：

```php
$itemData = array_filter($itemData, function ($value) {
    return $value !== null && strlen($value);
});
```

**注意：** `array_filter` 会移除 `null`、`''`、`false`，但**保留 `'0'` 和 `0`**。这意味着如果想显式设置某个字段为 `''`，会被过滤掉而不更新。

### 4.2 客户导入默认值

- `consent`：空值默认 `0`（[L419](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Customers.php#L419)）
- `taxable`：空值默认 `0`（[L451](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Customers.php#L451)）
- `discount`：空值默认 `0.00`（[Customer.php L273](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Models/Customer.php#L273)）
- `discount_type`：空值默认 `PERCENT`（[Customer.php L274](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Models/Customer.php#L274)）
- `date`：当前时间（[L452](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Customers.php#L452)）
- `employee_id`：当前登录用户（[L453](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Customers.php#L453)）

---

## 五、数据验证逻辑

### 5.1 商品导入验证

位于 `validateCSVData()` 方法（[L1157-L1252](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Items.php#L1157-L1252)）：

| 验证项 | 代码位置 | 说明 |
|-------|---------|------|
| 必填字段 | L1163-L1174 | name, category, unit_price（仅新增时） |
| 数字验证 | L1186-L1205 | cost_price, unit_price, reorder_level, supplier_id, 税率, 库存数量 |
| 库存位置 | L1208-L1212 | 校验 location_ 前缀的列名是否为合法位置 |
| 属性类型 | L1215-L1249 | DROPDOWN 检查选项值；DECIMAL 检查数字；DATE 检查日期格式 |
| 条码唯一性 | L1038-L1041 | 调用 `item_number_exists()` 检查重复 |
| 商品存在性 | L1178-L1183 | 更新时检查 item_id 是否存在 |

**特殊值 `_DELETE_`**（[L1221-L1223](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Items.php#L1221-L1223)）：
属性列填入 `_DELETE_` 表示删除该属性关联。

### 5.2 客户导入验证

- 邮箱格式验证（[L425-L429](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Customers.php#L425-L429)）
- 邮箱唯一性验证（[L458](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Customers.php#L458)）
- 账号唯一性验证（[L460-L463](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Customers.php#L460-L463)）

---

## 六、批量写入链路

### 6.1 商品导入 - 全链路事务

**事务范围**（[L1006-L1098](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Items.php#L1006-L1098)）：

```
事务开始 (transBegin)
    ↓
逐行处理：
    ├─ 字段映射与验证
    ├─ 保存商品基本信息 → [Item.php::save_value()](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Models/Item.php#L443-L468)
    ├─ 保存税务数据 → [Item_taxes.php::save_value()](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Models/Item_taxes.php#L36-L57)
    ├─ 保存库存数量 → [Item_quantity.php::save_value()](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Models/Item_quantity.php#L45-L57)
    ├─ 保存库存变更记录 → [Inventory.php::insert()](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Models/Inventory.php)
    └─ 保存属性数据 → [Attribute.php::saveCSVRowAttributeData()](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Models/Attribute.php#L940-L973)
    ↓
全部成功 → transCommit
有失败 → transRollback
```

**⚠️ 关键设计：全部成功或全部失败**

虽然代码逐行记录失败行号（`$failCodes[]`），但**任何一行失败都会导致整个事务回滚**，即：
- 100 行数据中第 50 行失败 → 前 49 行也会被撤销
- 返回消息提示"部分失败"，但实际上**没有任何数据被保存**

### 6.2 客户导入 - 无事务，逐行提交

**处理流程**（[L418-L487](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Customers.php#L418-L487)）：

```
逐行读取：
    ├─ 字段映射（按索引）
    ├─ 数据验证
    ├─ 保存客户数据 → [Customer.php::save_customer()](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Models/Customer.php#L213-L234)
    └─ 失败记录行号，成功继续下一行
    ↓
返回成功/失败统计
```

**⚠️ 数据不一致风险：**
- 100 行数据中第 50 行失败 → 前 49 行已永久保存
- 没有回滚机制，可能导致部分数据导入

---

## 七、部分失败处理对比

| 维度 | 商品导入 | 客户导入 |
|-----|---------|---------|
| 事务范围 | 全局事务，所有行 | 无事务，单行 |
| 失败影响 | 全部回滚 | 仅失败行跳过 |
| 数据一致性 | 高（原子性） | 低（部分成功） |
| 失败反馈 | 返回所有失败行号 | 返回所有失败行号 |
| 重试策略 | 修复后重新导入全部 | 仅需导入失败行 |

### 7.1 商品导入的"伪部分失败"

**代码问题**（[L1089-L1092](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Items.php#L1089-L1092)）：

```php
if (count($failCodes) > 0) {
    $message = lang('Items.csv_import_partially_failed', [...])
    $db->transRollback();  // 全部回滚！
    return ['success' => false, 'message' => $message];
}
```

**用户体验问题：** 消息说"部分失败"，但实际上**没有任何数据被保存**，用户可能误解为部分成功。

---

## 八、容易遗漏的关键点

### 8.1 字段映射层面

1. **商品导入硬编码列名**：修改 CSV 模板列名必须同步修改控制器代码，没有集中配置
2. **客户导入索引依赖**：CSV 列顺序严格固定，不能插入或删除列
3. **动态列前缀约定**：`location_` 和 `attribute_` 是隐含约定，无校验机制

### 8.2 默认值层面

4. **`array_filter` 陷阱**：空字符串 `''` 会被过滤，无法显式清空字段
5. **新增/更新差异化**：`allow_alt_description` 和 `is_serialized` 在新增和更新时空值处理逻辑不同
6. **库存数量默认值**：更新时空库存列被跳过（`continue`），新增时设为 0

### 8.3 事务与失败处理层面

7. **商品导入全回滚**：消息提示"部分失败"具有误导性，实际是全部失败
8. **客户导入无事务**：中间失败会导致数据不一致
9. **失败行号计算**：表头是第 1 行，数据从第 2 行开始，所以 `$key + 2`（[L1069](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Controllers/Items.php#L1069)）

### 8.4 数据一致性层面

10. **条码检查时机**：在 `save_value` 之前检查，但并发导入时仍可能出现重复
11. **属性值大小写不敏感**：`strcasecmp` 比较，自动更新为导入的大小写（[Attribute.php L902-L908](file:///d:/fz/0601-1/solo-dogfeeding/code/18-opensourcepos/app/Models/Attribute.php#L902-L908)）
12. **属性 `_DELETE_` 标记**：仅商品导入支持，客户导入无此机制

---

## 九、代码优化建议

### 9.1 商品导入事务消息

将"部分失败"改为更准确的表述，或改为真正的部分导入（逐行事务）。

### 9.2 客户导入列名匹配

改为按列名匹配，而非索引依赖，提高 CSV 格式灵活性。

### 9.3 字段映射集中配置

提取列名映射为配置数组，避免硬编码散落在代码各处。

### 9.4 空值语义明确

区分"不修改"（`null`）和"设为空"（`''`）的语义，避免 `array_filter` 误过滤。

### 9.5 统一失败处理策略

两个模块采用一致的事务策略，降低用户理解成本。
