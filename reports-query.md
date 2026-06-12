# OSPOS 报表过滤条件入口分支与参数流转差异分析

本文档针对 OSPOS 报表系统中三条核心路径——**通用汇总报表**、**折扣报表**、**仅按日期统计**——从路由入口、控制器方法签名、`$inputs` 组装、模型选择到查询构造的全链路差异进行逐段对照分析。

---

## 一、三条路径的整体定位

| 维度 | 路径 A：通用汇总报表 | 路径 B：折扣报表 | 路径 C：仅按日期统计 |
|------|--------------------|-----------------|---------------------|
| 代表报表 | `summary_sales`、`summary_items`、`summary_categories`、`summary_customers`、`summary_suppliers`、`summary_employees`、`summary_taxes`、`summary_sales_taxes` | `summary_discounts`、`specific_discounts` | `summary_payments`、`summary_expenses_categories` |
| 输入表单方法 | [Reports::date_input()](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Controllers/Reports.php#L655-L666) | [Reports::summary_discounts_input()](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Controllers/Reports.php#L537-L549) | [Reports::date_input_only()](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Controllers/Reports.php#L674-L680) |
| 核心特征 | 标准四参模式（日期+销售类型+仓库） | 标准四参 + `discount_type`（折扣类型） | 最少参数，控制器层硬编码默认值 |
| URL 段数 | 5 段（/summary_xxx/start/end/sale_type/location_id） | 6 段（/summary_discounts/start/end/sale_type/location_id/discount_type） | 3~4 段（/summary_payments/start/end） |

---

## 二、路由层：入口分支匹配的不一致

路由定义在 [Routes.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Config/Routes.php#L18-L42)，三条路径的匹配优先级如下（从上到下匹配，先匹配先命中）：

```php
// ========== 路径 C：仅按日期统计（最高优先级特例路由） ==========
$routes->add('reports/summary_expenses_categories', 'Reports::date_input_only');
$routes->add('reports/summary_payments',            'Reports::date_input_only');

// ========== 路径 B：折扣报表（次高优先级特例路由） ==========
$routes->add('reports/summary_discounts',           'Reports::summary_discounts_input');

// ========== 路径 A：通用汇总报表（最低优先级通配路由，兜底所有其他 summary_xxx） ==========
$routes->add('reports/summary_(:any)',              'Reports::date_input');
```

### 2.1 匹配规则的微妙之处

**不一致点 1：同属 `summary_` 前缀，却走不同输入表单**

三个 URL 外观几乎完全一致：

| 用户访问 URL | 命中路由 | 调用控制器方法 |
|-------------|---------|--------------|
| `reports/summary_sales` | 通配 `summary_(:any)` | `date_input()` |
| `reports/summary_discounts` | 特例 `summary_discounts` | `summary_discounts_input()` |
| `reports/summary_payments` | 特例 `summary_payments` | `date_input_only()` |

**不一致点 2：URL 通配模式的参数占位数量不匹配**

```php
// 通用汇总：通配路由只占 3 段参数（$2/$3/$4），但控制器方法有 4 个形参（location_id 有默认值）
$routes->add('reports/summary_(:any)/(:any)/(:any)', 'Reports::Summary_$1/$2/$3/$4');
//                                                    ↑ 实际上传了 3 个参数给一个需要 4~5 个参数的方法
```

- 对于路径 A（4 参方法）：第 4 个参数 `location_id` 靠默认值 `'all'` 兜底，当 URL 未传 location_id 时不会报错
- 对于路径 B（5 参方法 `summary_discounts`）：URL 必须传 5 个段，第 5 个 `discount_type` 靠默认值 `0` 兜底
- 对于路径 C（2~3 参方法）：路由层没有单独的带参路由，实际靠 CodeIgniter 的自动参数绑定，需要访问完整的 URL 如 `reports/summary_payments/start/end`

---

## 三、控制器层：方法签名与参数接收差异

### 3.1 方法签名对照表

| 路径 | 控制器方法 | 签名 | 必填参数数 | 可选参数及默认值 |
|------|----------|------|-----------|----------------|
| **A 通用汇总** | `summary_sales()` | [L131](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Controllers/Reports.php#L131) `(string $start_date, string $end_date, string $sale_type, string $location_id = 'all')` | 3 | 1 个 (`location_id='all'`) |
| **A 通用汇总** | `summary_items()`、`summary_categories()`、`summary_customers()`、`summary_suppliers()`、`summary_employees()`、`summary_taxes()`、`summary_sales_taxes()` | 同上（与 summary_sales 完全一致） | 3 | 1 个 (`location_id='all'`) |
| **B 折扣报表** | `summary_discounts()` | [L555](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Controllers/Reports.php#L555) `(string $start_date, string $end_date, string $sale_type, string $location_id = 'all', int $discount_type = 0)` | 3 | 2 个 (`location_id='all'`, `discount_type=0`) |
| **C 仅日期** | `summary_payments()` | [L594](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Controllers/Reports.php#L594) `(string $start_date, string $end_date)` | **2** | **0 个** |
| **C 仅日期** | `summary_expenses_categories()` | [L223](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Controllers/Reports.php#L223) `(string $start_date, string $end_date, string $sale_type)` | 3 | 0 个 |
| **B 折扣（明细）** | `specific_discounts()` | [L1524](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Controllers/Reports.php#L1524) `(string $start_date, string $end_date, string $discount, string $sale_type, string $discount_type)` | **5** | 0 个（全必填） |

### 3.2 `summary_payments` 的特殊之处：参数在控制器层硬编码

```php
// 路径 C：summary_payments - 控制器内硬编码 sale_type 和 location_id
public function summary_payments(string $start_date, string $end_date): string
{
    $inputs = [
        'start_date'  => $start_date,
        'end_date'    => $end_date,
        'sale_type'   => 'complete',   // ← 硬编码！用户没有选择余地
        'location_id' => 'all'         // ← 硬编码！用户没有选择余地
    ];
}
```

**不一致点 3：`summary_payments` 与其他报表设计模式完全背离**

- 其他所有销售类报表：`sale_type` 和 `location_id` 来自用户输入，URL 传递
- `summary_payments`：`sale_type` 固定为 `'complete'`、`location_id` 固定为 `'all'`，用户在输入表单上也看不到这两个选项

对应的输入表单 `date_input_only()` 也只传了一个空数组给视图：

```php
// 路径 C 的输入表单
public function date_input_only(): string
{
    $this->clearCache();
    $data = [];                              // ← 没有 stock_locations、没有 sale_type_options
    return view('reports/date_input', $data);
}
```

而通用汇总的 `date_input()` 则完整注入了所有选项：

```php
// 路径 A 的输入表单
public function date_input(): string
{
    $stock_locations = $data = $this->stock_location->get_allowed_locations('sales');
    $stock_locations['all'] = lang('Reports.all');
    $data['stock_locations']   = array_reverse($stock_locations, true);
    $data['mode']              = 'sale';
    $data['sale_type_options'] = $this->get_sale_type_options();
    return view('reports/date_input', $data);
}
```

### 3.3 `summary_expenses_categories` 的特殊之处：有 sale_type 但无 location_id

```php
public function summary_expenses_categories(string $start_date, string $end_date, string $sale_type): string
{
    $inputs = ['start_date' => $start_date, 'end_date' => $end_date, 'sale_type' => $sale_type];
    //                                                             ↑ 没有 location_id
}
```

原因：费用（expenses）是企业内部支出，不涉及仓库库存位置，业务上合理。但输入表单也走 `date_input_only()`，表单上既没有仓库下拉也没有销售类型下拉——`sale_type` 参数的来源需要依赖前端 JS 的默认值。

---

## 四、视图层：前端表单字段与 URL 拼接差异

视图 [date_input.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Views/reports/date_input.php) 是三条路径共用的，但通过控制器注入不同的 `$data` 变量来控制字段的显示。

### 4.1 表单字段显示的条件判断

| 字段 | 显示条件 | 路径 A（通用汇总） | 路径 B（折扣报表） | 路径 C（仅日期） |
|------|---------|-------------------|-------------------|-----------------|
| **日期范围** | 始终显示 | ✅ 显示 | ✅ 显示 | ✅ 显示 |
| **销售类型下拉** | `!empty($mode) && $mode == 'sale'` | ✅ 显示 (`mode='sale'`) | ✅ 显示 (`mode='sale'`) | ❌ 不显示（`mode` 未注入） |
| **折扣类型下拉** | `isset($discount_type_options)` | ❌ 不显示 | ✅ 显示（`discount_type_options` 注入了百分比/固定金额） | ❌ 不显示 |
| **仓库位置下拉** | `!empty($stock_locations) && count > 2` | ✅ 显示（>1 仓库时） | ✅ 显示（>1 仓库时） | ❌ 不显示（`stock_locations` 未注入） |

### 4.2 URL 拼接 JS 代码的不一致

**路径 A 和路径 B 共用同一段 JS**（[date_input.php#L95-L97](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Views/reports/date_input.php#L95-L97)）：

```javascript
$("#generate_report").click(function() {
    //                   段1         段2       段3                     段4                         段5
    window.location = [window.location, start_date, end_date, $("#input_type").val() || 0, $("#location_id").val() || 'all', $("#discount_type_id").val() || 0].join("/");
});
```

这段代码**一次性拼接了 5 个 URL 段**，但三条路径实际只需要其中不同的子集：

| 路径 | 需要的段数 | 实际拼接段数 | 多余段的处理 |
|------|-----------|------------|------------|
| 路径 A（通用汇总） | 4 段（start/end/sale_type/location_id） | 5 段 | 第 5 段 `discount_type=0` 被拼上了，但控制器方法签名中没有该参数。CodeIgniter 会忽略多余的 URL 参数，不会报错，但 URL 中会出现一个无意义的 `/0` |
| 路径 B（折扣报表） | 5 段（start/end/sale_type/location_id/discount_type） | 5 段 | 完全匹配，没有问题 |
| 路径 C（仅日期） | 2~3 段 | 5 段 | **严重问题**：`$mode` 为空时前端不渲染 `#input_type`，`$("#input_type").val()` 返回 `undefined`，被 `|| 0` 兜底为 `0`，URL 变成 `/start/end/0/all/0`，但 `summary_payments` 只有 2 个形参，多余的 `/0/all/0` 被忽略；`summary_expenses_categories` 有 3 个形参，`sale_type` 会接收到 `0`（而不是合法的 `'complete'`/`'sales'` 等），可能导致模型层查询异常 |

**不一致点 4：JS 硬编码拼接 5 段，但各路径方法签名参数个数不一致**——这是三条路径参数流转不一致的根因之一。

而 `specific_discounts` 明细路径使用另一个视图 [specific_input.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Views/reports/specific_input.php)，有完全不同的拼接逻辑：

```javascript
// specific_input.php#L89-L96
$("#generate_report").click(function() {
    var specific_input_data = $('#specific_input_data').val();
    if (!$(".discount_percent").is(":visible")) {
        specific_input_data = $('#discount_fixed').val();  // 根据折扣类型切换下拉框 / 数字输入框
    }
    //                    段1         段2       段3                   段4                        段5
    window.location = [window.location, start_date, end_date, specific_input_data, $("#input_type").val() || 0, $("#discount_type_id").val() || 0].join("/");
});
```

注意 `specific_input.php` 中 URL 段 3 的位置是**折扣值**（而非通用报表的 sale_type），段 4 才是 sale_type，段 5 是 discount_type。对应控制器方法签名中参数顺序是 `($start_date, $end_date, $discount, $sale_type, $discount_type)`，完全一致。

---

## 五、`$inputs` 数组组装差异：传给模型的字段对照

### 5.1 `$inputs` 字段矩阵

| `$inputs` 字段 | 路径 A `summary_sales` | 路径 B `summary_discounts` | 路径 B `specific_discounts` | 路径 C `summary_payments` | 路径 C `summary_expenses_categories` |
|---------------|----------------------|---------------------------|----------------------------|--------------------------|-------------------------------------|
| `start_date` | ✅ URL 参数 | ✅ URL 参数 | ✅ URL 参数 | ✅ URL 参数 | ✅ URL 参数 |
| `end_date` | ✅ URL 参数 | ✅ URL 参数 | ✅ URL 参数 | ✅ URL 参数 | ✅ URL 参数 |
| `sale_type` | ✅ URL 参数 | ✅ URL 参数 | ✅ URL 参数（段 4） | ❌ **硬编码 `'complete'`** | ✅ URL 参数 |
| `location_id` | ✅ URL 参数 / 默认 `'all'` | ✅ URL 参数 / 默认 `'all'` | ❌ **不存在** | ❌ **硬编码 `'all'`** | ❌ **不存在** |
| `discount_type` | ❌ **不存在** | ✅ URL 参数 / 默认 `0` | ✅ URL 参数（段 5） | ❌ **不存在** | ❌ **不存在** |
| `discount`（折扣值） | ❌ 不存在 | ❌ 不存在 | ✅ URL 参数（段 3） | ❌ 不存在 | ❌ 不存在 |
| `customer_id` / `employee_id` / `supplier_id` | ❌ 不存在 | ❌ 不存在 | ❌ 不存在（specific_* 其他报表才有） | ❌ 不存在 | ❌ 不存在 |
| `definition_ids`（自定义属性） | ❌ 不存在（只有 detailed 报表有） | ❌ 不存在 | ❌ 不存在 | ❌ 不存在 | ❌ 不存在 |

### 5.2 三条路径的 `$inputs` 组装代码对照

```php
// ===== 路径 A：通用汇总（如 summary_sales L135-L140） =====
$inputs = [
    'start_date'  => $start_date,
    'end_date'    => $end_date,
    'sale_type'   => $sale_type,
    'location_id' => $location_id
];


// ===== 路径 B：折扣汇总（summary_discounts L559-L565） =====
$inputs = [
    'start_date'    => $start_date,
    'end_date'      => $end_date,
    'sale_type'     => $sale_type,
    'location_id'   => $location_id,
    'discount_type' => $discount_type    // ← 多了这一个
];


// ===== 路径 B：折扣明细（specific_discounts L1528-L1534） =====
$inputs = [
    'start_date'    => $start_date,
    'end_date'      => $end_date,
    'discount'      => $discount,        // ← 折扣值，代替了 location_id 的位置
    'sale_type'     => $sale_type,
    'discount_type' => $discount_type
];
// 注意：没有 location_id


// ===== 路径 C：仅日期统计 - summary_payments（L598-L603） =====
$inputs = [
    'start_date'  => $start_date,
    'end_date'    => $end_date,
    'sale_type'   => 'complete',         // ← 硬编码！不是 URL 参数
    'location_id' => 'all'               // ← 硬编码！不是 URL 参数
];


// ===== 路径 C：仅日期统计 - summary_expenses_categories（L227） =====
$inputs = ['start_date' => $start_date, 'end_date' => $end_date, 'sale_type' => $sale_type];
// 注意：没有 location_id，也没有 discount_type
```

**不一致点 5：`summary_payments` 虽然 `$inputs` 有 4 个字段和通用汇总一样，但其中两个是硬编码的**，导致模型层查询行为实际上被锁定为「全部仓库 + 已完成销售」，无法按用户输入变化。

---

## 六、模型选择与查询构造差异

### 6.1 模型继承关系与选择

| 路径 | 报表 | 控制器中实例化方式 | 模型类 | 继承关系 |
|------|------|-----------------|--------|---------|
| A 通用汇总 | `summary_sales` | 构造函数 `$this->summary_sales = model(Summary_sales::class)` | [Summary_sales](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_sales.php) | extends Summary_report |
| A 通用汇总 | `summary_items` | 同上 | [Summary_items](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_items.php) | extends Summary_report |
| A 通用汇总 | `summary_taxes` | 同上 | [Summary_taxes](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_taxes.php) | extends Summary_report（但完全重写了查询） |
| B 折扣汇总 | `summary_discounts` | 同上 | [Summary_discounts](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_discounts.php) | extends Summary_report（但完全重写了查询） |
| B 折扣明细 | `specific_discounts` | 方法内动态 `model(Specific_discount::class)` | [Specific_discount](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Specific_discount.php) | extends Report（不使用 Summary_report 模板） |
| C 仅日期 | `summary_payments` | 构造函数 `$this->summary_payments = model(Summary_payments::class)` | [Summary_payments](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_payments.php) | extends Summary_report（但完全重写了查询） |
| C 仅日期 | `summary_expenses_categories` | 同上 | [Summary_expenses_categories](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_expenses_categories.php) | extends Summary_report（但完全重写了查询） |

### 6.2 查询构造模式对照

#### 路径 A：通用汇总报表（使用 Summary_report 模板方法）

以 `Summary_sales` 为例，继承 [Summary_report::getData()](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_report.php#L111-L173) 的模板方法流程：

```
getData($inputs)
  ├── 建 2 张临时表（sales_items_taxes_temp、sales_payments_temp）
  ├── $builder = db->table('sales_items')
  ├── _select()  → parent::__common_select() 计算 subtotal/tax/total/cost/profit
  ├── _from()    → parent::__common_from() JOIN sales + 临时表
  ├── _where()   → parent::__common_where() 处理 $inputs 中 4 个字段：
  │                  start_date / end_date（日期范围）
  │                  location_id（仓库过滤）
  │                  sale_type（sale_status + sale_type 组合过滤）
  ├── _group_order() → 子类定义 GROUP BY（Summary_sales 按 sale_date）
  └── getResultArray()
```

`__common_where` 中对 `location_id` 的判断：

```php
if ($inputs['location_id'] != 'all') {           // ← 只有非 'all' 时才加 WHERE 条件
    $builder->where('sales_items.item_location', $inputs['location_id']);
}
```

对 `sale_type` 的判断（6 个分支）：

```php
switch ($inputs['sale_type']) {
    case 'complete':  // sale_status=COMPLETED + sale_type IN (POS, INVOICE, RETURN)
    case 'sales':     // sale_status=COMPLETED + sale_type IN (POS, INVOICE)
    case 'quotes':    // sale_status=SUSPENDED + sale_type=QUOTE
    case 'work_orders': // sale_status=SUSPENDED + sale_type=WORK_ORDER
    case 'canceled':  // sale_status=CANCELED
    case 'returns':   // sale_status=COMPLETED + sale_type=RETURN
}
```

**路径 A 的查询 `$inputs` 字段依赖完整：4 个字段全部参与 WHERE 条件构造。**

#### 路径 B：折扣汇总报表（完全重写查询，不使用模板）

[Summary_discounts::getData()](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_discounts.php#L19-L157) 完全重写，不调用基类的 `_select/_from/_where/_group_order`：

```php
public function getData(array $inputs): array
{
    // 动态拼接 SELECT：根据 discount_type (0=PERCENT / 1=FIXED) 切换显示方式
    if ($inputs['discount_type'] == PERCENT) {
        $builder->select("CONCAT(discount, '%') AS discount, ...");
    } else {
        $builder->select("CONCAT('$', discount) AS discount, ...");
    }

    // WHERE 条件使用了 $inputs 中的 5 个字段：
    $builder->where('discount_type', $inputs['discount_type']);   // ← 折扣类型必须有
    // 日期范围
    // location_id（如果不是 'all'）
    // sale_type（6 个分支，与通用汇总完全相同的 switch）

    // GROUP BY：按折扣值分组（不是按日期/商品/客户）
    $builder->groupBy('discount');
    $builder->orderBy('discount');
}
```

**路径 B（折扣汇总）查询 `$inputs` 字段依赖：5 个字段全部使用，其中 `discount_type` 同时影响 SELECT 显示格式和 WHERE 过滤条件。**

[Specific_discount::getData()](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Specific_discount.php#L71-L160) 则走临时表模式：

```php
public function getData(array $inputs): array
{
    // 先调用 Sale::create_temp_table($inputs) 建 sales_items_temp 宽表

    $builder = db->table('sales_items_temp');
    $builder->select(...);

    // WHERE 条件使用 $inputs 中的字段：
    $builder->where('discount >=', $inputs['discount']);       // ← 折扣值：大于等于某个值
    $builder->where('discount_type', $inputs['discount_type']); // ← 折扣类型精确匹配
    // sale_type（6 个分支）
    // 注意：没有 location_id 条件（因为 $inputs 中本来就没有）

    // GROUP BY sale_id，然后 N+1 查询每个 sale 的明细
}
```

**不一致点 6：`Specific_discount` 的 `$inputs` 没有 `location_id` 字段，查询结果不按仓库过滤**——这与 `Detailed_sales` / `Specific_customer` 等明细报表不同（它们有 `location_id`）。

#### 路径 C：仅日期统计报表

**`Summary_payments`** [getData()](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_payments.php#L100-L279) 完全重写，创建 **3 张专用临时表**：

```php
public function getData(array $inputs): array
{
    // 步骤 1：建 sumpay_sales_temp（按 sale_id 汇总的销售临时表）
    //   WHERE 条件：
    //     - 日期范围（start_date / end_date）
    //     - $inputs['sale_type']：但控制器已经硬编码为 'complete'
    //     - $inputs['location_id']：但控制器已经硬编码为 'all'，所以 location 条件被跳过

    // 步骤 2：建 sumpay_items_temp（按 sale_id + 支付方式汇总的明细临时表）
    // 步骤 3：建 sumpay_taxes_temp（税额汇总临时表）
    // 步骤 4：UPDATE sumpay_items_temp SET trans_amount += 税额 （相关子查询更新）

    // 步骤 5：查询销售类型统计段（UNION ALL 风格，PHP 层合并）
    // 步骤 6：查询支付方式统计段
    // 步骤 7：两段数据之间插入一行 '<HR>' 分隔行
}
```

**路径 C（`summary_payments`）查询 `$inputs` 字段依赖：虽然定义了 4 个字段，但 `sale_type` 和 `location_id` 实际上永远是硬编码值，查询行为无法变化。**

**`Summary_expenses_categories`** [getData()](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_expenses_categories.php#L27-L95) 最简单：

```php
public function getData(array $inputs): array
{
    // 直接查 expenses + expense_categories 两表
    $builder = db->table('expenses');
    $builder->join('expense_categories', 'expenses.category_id = expense_categories.id');
    $builder->select('category_name, COUNT(*) as count, SUM(amount) AS total_amount, SUM(tax_amount) AS total_tax_amount');

    // WHERE 条件：只有日期范围
    if (empty($this->config['date_or_time_format'])) {
        $builder->where('DATE(expense_date) BETWEEN ...');
    } else {
        $builder->where('expense_date BETWEEN ...');
    }
    // 注意：$inputs['sale_type'] 完全没被使用！！

    $builder->groupBy('expenses.category_id');
}
```

**不一致点 7：`summary_expenses_categories` 的 `$inputs` 有 `sale_type` 字段，但模型查询中完全忽略它**——`sale_type` 对费用报表毫无意义（费用不是销售），但控制器方法签名和 `$inputs` 数组仍然保留了这个字段，属于冗余参数。

### 6.3 `getSummaryData` 方法的字段依赖差异

| 模型 | `getSummaryData` 使用的 `$inputs` 字段 |
|------|--------------------------------------|
| Summary_report 基类（路径 A 大部分报表） | `start_date`, `end_date`, `sale_type`, `location_id` |
| Summary_discounts | `start_date`, `end_date`, `sale_type`, `location_id`, `discount_type`（5 个全用） |
| Specific_discount | `discount`, `discount_type`, `sale_type`（没有 location_id） |
| Summary_payments | `start_date`, `end_date`（只有 2 个，因为 sale_type/location_id 被硬编码） |
| Summary_expenses_categories | `start_date`, `end_date`（sale_type 被忽略） |

---

## 七、全链路参数流转对照表（一张表看完）

```
┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                           路径 A：通用汇总（以 summary_sales 为例）                                        │
├─────────────────┬────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ 路由入口         │ reports/summary_sales         → Reports::date_input()                                                     │
│ 带参路由         │ reports/summary_sales/a/b/c   → Reports::Summary_sales(a, b, c)   （location_id 靠默认值 'all'）          │
│ 带参路由（完整）   │ reports/summary_sales/a/b/c/d → Reports::Summary_sales(a, b, c, d)                                        │
│ 方法签名         │ (start, end, sale_type, location_id='all')  —— 4 个参数                                                     │
│ 输入表单注入字段   │ stock_locations, mode='sale', sale_type_options                                                          │
│ 前端 JS 拼接段数  │ 5 段（第 5 段 discount_type=0 被忽略）                                                                    │
│ $inputs 字段     │ start_date, end_date, sale_type, location_id  —— 4 个字段                                                   │
│ 实例化模型       │ Summary_sales （构造函数预加载）                                                                           │
│ 模型继承         │ extends Summary_report                                                                                    │
│ 查询构造模式     │ 模板方法 _select/_from/_where/_group_order                                                                │
│ 查询实际用字段   │ 4 个字段全部使用                                                                                           │
│ WHERE 过滤范围    │ 日期 + 仓库 + 销售类型                                                                                    │
└─────────────────┴────────────────────────────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                           路径 B-汇总：折扣报表（summary_discounts）                                        │
├─────────────────┬────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ 路由入口         │ reports/summary_discounts          → Reports::summary_discounts_input()                                   │
│ 带参路由         │ reports/summary_discounts/a/b/c/d/e → Reports::Summary_discounts(a, b, c, d, e)                           │
│ 方法签名         │ (start, end, sale_type, location_id='all', discount_type=0)  —— 5 个参数                                   │
│ 输入表单注入字段   │ stock_locations, mode='sale', sale_type_options, discount_type_options                                   │
│ 前端 JS 拼接段数  │ 5 段（完全匹配）                                                                                         │
│ $inputs 字段     │ start_date, end_date, sale_type, location_id, discount_type  —— 5 个字段                                   │
│ 实例化模型       │ Summary_discounts （构造函数预加载）                                                                       │
│ 模型继承         │ extends Summary_report（但完全重写查询）                                                                    │
│ 查询构造模式     │ 完全自定义，按折扣值 GROUP BY                                                                              │
│ 查询实际用字段   │ 5 个字段全部使用，discount_type 同时影响 SELECT 显示和 WHERE 过滤                                             │
│ WHERE 过滤范围    │ 日期 + 仓库 + 销售类型 + 折扣类型                                                                         │
└─────────────────┴────────────────────────────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                           路径 B-明细：折扣报表（specific_discounts）                                        │
├─────────────────┬────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ 路由入口         │ reports/specific_discounts       → Reports::specific_discount_input()                                     │
│ 带参路由         │ reports/specific_discounts/a/b/c/d/e → Reports::Specific_discounts(a, b, c, d, e)                         │
│ 方法签名         │ (start, end, discount, sale_type, discount_type)  —— 5 个参数（全必填，无默认值）                            │
│ 输入表单注入字段   │ specific_input_data, specific_input_name, sale_type_options, discount_type_options                        │
│ 前端视图         │ specific_input.php（有折扣类型切换 JS：百分比下拉 / 固定金额数字框）                                         │
│ 前端 JS 拼接段数  │ 5 段（段 3 是折扣值，段 4 是 sale_type，段 5 是 discount_type）                                           │
│ $inputs 字段     │ start_date, end_date, discount, sale_type, discount_type  —— 5 个字段（注意：没有 location_id）             │
│ 实例化模型       │ model(Specific_discount::class)（方法内动态实例化）                                                         │
│ 模型继承         │ extends Report                                                                                            │
│ 查询构造模式     │ 临时表模式（Sale::create_temp_table + sales_items_temp 宽表 + N+1 查询）                                    │
│ 查询实际用字段   │ discount（>=）、discount_type（=）、sale_type，**没有 location_id**                                        │
│ WHERE 过滤范围    │ 日期 + 折扣值 + 折扣类型 + 销售类型（**不按仓库过滤**）                                                    │
└─────────────────┴────────────────────────────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                           路径 C-1：仅日期（summary_payments）                                                │
├─────────────────┬────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ 路由入口         │ reports/summary_payments         → Reports::date_input_only()                                              │
│ 带参访问         │ reports/summary_payments/a/b      → Reports::Summary_payments(a, b)                                         │
│ 方法签名         │ (start, end)  —— 2 个参数（全必填）                                                                        │
│ 输入表单注入字段   │ 空数组 []                                                                                                │
│ 前端 JS 拼接段数  │ 5 段（但 sale_type=0, location_id=all, discount_type=0 都是假的，被控制器忽略）                             │
│ $inputs 字段     │ start_date, end_date, sale_type='complete', location_id='all'  —— 4 个字段，2 个硬编码                      │
│ 实例化模型       │ Summary_payments （构造函数预加载）                                                                         │
│ 模型继承         │ extends Summary_report（但完全重写查询）                                                                    │
│ 查询构造模式     │ 3 张临时表 + UPDATE 相关子查询 + PHP 层数据合并                                                              │
│ 查询实际用字段   │ start_date, end_date（sale_type/location_id 虽然被使用，但值被硬编码锁定）                                    │
│ WHERE 过滤范围    │ 仅日期范围（仓库=全部，销售类型=已完成，均硬编码）                                                          │
└─────────────────┴────────────────────────────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                           路径 C-2：仅日期（summary_expenses_categories）                                    │
├─────────────────┬────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ 路由入口         │ reports/summary_expenses_categories → Reports::date_input_only()                                           │
│ 带参访问         │ reports/summary_expenses_categories/a/b/c → Reports::Summary_expenses_categories(a, b, c)                  │
│ 方法签名         │ (start, end, sale_type)  —— 3 个参数（全必填）                                                              │
│ 输入表单注入字段   │ 空数组 []                                                                                                │
│ 前端 JS 拼接段数  │ 5 段（段 3=sale_type 会收到 0，段 4-5 被忽略）                                                              │
│ $inputs 字段     │ start_date, end_date, sale_type  —— 3 个字段                                                               │
│ 实例化模型       │ Summary_expenses_categories （构造函数预加载）                                                              │
│ 模型继承         │ extends Summary_report（但完全重写查询）                                                                    │
│ 查询构造模式     │ 直接查 expenses + expense_categories 两表                                                                  │
│ 查询实际用字段   │ 只用 start_date, end_date（sale_type 被完全忽略）                                                           │
│ WHERE 过滤范围    │ 仅日期范围                                                                                                │
└─────────────────┴────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 八、已发现的不一致问题汇总

以下是三条路径在参数流转与模型选择中的**7 处实质不一致**：

| 编号 | 问题 | 涉及路径 | 说明 |
|------|------|---------|------|
| 1 | 同前缀路由走不同输入表单 | 全部 | `summary_sales`/`summary_discounts`/`summary_payments` 三个 URL 外观一致，但分别命中 `date_input()` / `summary_discounts_input()` / `date_input_only()`，代码阅读者很难从 URL 预判行为差异 |
| 2 | 通配路由参数占位数与方法签名参数个数不匹配 | 全部 | 路由 `summary_(:any)/(:any)/(:any)` 只传 3 段，但方法签名需要 4~5 个参数，靠默认值兜底 |
| 3 | `summary_payments` 硬编码 `sale_type` 和 `location_id` | 路径 C-1 | 用户无法通过输入表单控制这两个维度，与其他所有销售类报表的交互模式不一致 |
| 4 | `date_input.php` 的 JS 固定拼接 5 个 URL 段 | 全部 | 三条路径各需要 2~5 段，但 JS 永远拼 5 段，导致路径 C 收到 `sale_type=0` 这样的非法值 |
| 5 | `summary_payments` 的 `$inputs` 字段半硬编码 | 路径 C-1 | 虽然有 4 个字段，但 2 个是写死的，模型层行为被锁定，参数名存在但无实际控制作用 |
| 6 | `Specific_discount` 没有 `location_id` 过滤 | 路径 B-明细 | 其他明细报表（`Detailed_sales`、`Specific_customer`）都有仓库维度过滤，唯独折扣明细没有，导致同仓库的用户可能看到其他仓库的折扣销售数据 |
| 7 | `summary_expenses_categories` 的 `sale_type` 参数冗余 | 路径 C-2 | `$inputs` 和方法签名都有 `sale_type`，但模型查询完全不使用，属于"僵尸参数"，易误导维护者 |

---

## 九、模型层查询性能影响的差异

| 路径 | 临时表数量 | JOIN 数 | GROUP BY 维度 | 大范围日期下的性能风险 |
|------|-----------|---------|--------------|---------------------|
| 路径 A（通用汇总） | 2 张（MEMORY 引擎税表 + 支付表） | 4~6 表 | 日期/商品/品类/客户/供应商/员工 | 中等：单次聚合查询，可利用索引 |
| 路径 B（折扣汇总） | 2 张（同路径 A） | 4 表 | 折扣值（通常只有 10 个以内离散值） | 低：折扣值分组后数据量很小 |
| 路径 B（折扣明细） | 3 张（含 sales_items_temp 宽表） | 7~9 表（建宽表时） + N+1 查询 | sale_id（可能上万行） | 极高：N+1 查询，10000 笔销售会产生 20001 次查询 + 无仓库过滤导致查全部仓库数据 |
| 路径 C（summary_payments） | **3 张专用临时表** | 3~4 表 + UPDATE 相关子查询 | 销售类型 + 支付方式（通常 10 行以内） | 中高：UPDATE 语句使用相关子查询，临时表大时性能差 |
| 路径 C（费用分类） | 0 张 | 2 表 | 费用分类（通常几十个） | 极低：最简单的查询，无临时表 |
