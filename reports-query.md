# OSPOS 报表查询构造代码分析

## 一、整体架构

OSPOS 报表系统采用三层架构：**控制器层 → 模型层 → 视图层**。

```
路由请求 → Reports 控制器 → 报表模型（Report/Summary_report 继承体系）→ 视图
```

### 1.1 文件分布

| 层级 | 路径 | 说明 |
|------|------|------|
| 控制器 | [Reports.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Controllers/Reports.php) | 统一入口，所有报表请求的调度中心 |
| 基类模型 | [Report.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Report.php) | 所有报表的抽象基类，定义接口契约 |
| 汇总基类 | [Summary_report.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_report.php) | 销售类汇总报表的模板方法基类 |
| 具体模型 | `app/Models/Reports/*.php` | 22 个具体报表模型 |
| 辅助模型 | [Sale.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Sale.php) | 提供销售临时表创建（create_temp_table） |
| 辅助模型 | [Receiving.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Receiving.php) | 提供收货临时表创建 |
| 视图 | `app/Views/reports/` | 输入表单、表格展示、图形展示 |

### 1.2 报表模型分类体系

```
Report (抽象基类)
├── Summary_report (销售汇总模板基类)
│   ├── Summary_sales              (按日期汇总销售)
│   ├── Summary_categories         (按品类汇总)
│   ├── Summary_items              (按商品汇总)
│   ├── Summary_customers          (按客户汇总)
│   ├── Summary_suppliers          (按供应商汇总)
│   ├── Summary_employees          (按员工汇总)
│   ├── Summary_taxes              (按税率汇总 - 重写了整个查询)
│   ├── Summary_sales_taxes        (按税务管辖区汇总 - 重写了整个查询)
│   ├── Summary_discounts          (按折扣汇总 - 重写了整个查询)
│   ├── Summary_payments           (按支付方式汇总 - 重写了整个查询)
│   └── Summary_expenses_categories (费用分类汇总 - 重写了整个查询)
├── Detailed_sales                 (销售明细表)
├── Detailed_receivings            (收货明细表)
├── Specific_customer              (指定客户明细)
├── Specific_employee              (指定员工明细)
├── Specific_discount              (指定折扣明细)
├── Specific_supplier              (指定供应商明细)
├── Inventory_summary              (库存汇总)
└── Inventory_low                  (库存告警)
```

---

## 二、过滤条件从输入到查询的完整路径

### 2.1 路由层：URL 参数映射

路由定义在 [Routes.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Config/Routes.php#L18-L42)：

```php
$routes->add('reports/summary_(:any)/(:any)/(:any)', 'Reports::Summary_$1/$2/$3/$4');
$routes->add('reports/summary_(:any)', 'Reports::date_input');          // 先展示输入表单
$routes->add('reports/detailed_(:any)/(:any)/(:any)/(:any)', 'Reports::Detailed_$1/$2/$3/$4');
$routes->add('reports/specific_(:any)/(:any)/(:any)/(:any)', 'Reports::Specific_$1/$2/$3/$4');
```

**规律**：
- 首次访问 `reports/summary_xxx` → 渲染输入表单 `date_input()`
- 表单提交后 URL 变为 `reports/summary_xxx/{start_date}/{end_date}/{sale_type}/{location_id}` → 调用具体报表方法

### 2.2 视图层：输入表单构造

输入表单在 [date_input.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Views/reports/date_input.php) 中，前端 JS 组装 URL：

```javascript
$("#generate_report").click(function() {
    window.location = [window.location, start_date, end_date, 
        $("#input_type").val() || 0, 
        $("#location_id").val() || 'all', 
        $("#discount_type_id").val() || 0].join("/");
});
```

**可传递的过滤参数**：

| 参数 | 来源 | 说明 |
|------|------|------|
| `start_date` / `end_date` | 日期选择器 | 时间范围，URL 编码格式 |
| `sale_type` | 下拉框 | `complete` / `sales` / `quotes` / `work_orders` / `canceled` / `returns` |
| `location_id` | 库存位置下拉 | 具体 location_id 或 `all` |
| `discount_type` | 折扣类型下拉 | `0`(百分比) / `1`(固定金额) |
| `customer_id` / `employee_id` / `supplier_id` | 特定对象选择 | specific 系列报表专用 |
| `payment_type` | 支付方式 | specific_customer 专用 |
| `receiving_type` | 收货类型 | detailed_receivings 专用 |
| `item_count` | 库存数量过滤 | inventory_summary 专用 |

### 2.3 控制器层：参数接收与组装

控制器方法签名示例（[Reports.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Controllers/Reports.php#L131-L168)）：

```php
public function summary_sales(string $start_date, string $end_date, string $sale_type, string $location_id = 'all'): string
{
    $inputs = [
        'start_date'  => $start_date,
        'end_date'    => $end_date,
        'sale_type'   => $sale_type,
        'location_id' => $location_id
    ];

    $report_data = $this->summary_sales->getData($inputs);
    $summary     = $this->summary_sales->getSummaryData($inputs);
    // ... 格式化数据后传视图
}
```

关键点：
- 所有参数都被打包进 `$inputs` 数组传给模型
- `sale_type_options` 通过 `get_sale_type_options()` 方法根据配置动态生成（如发票启用时才显示 quotes/work_orders）
- 所有报表方法都先调用 `clearCache()` 禁用浏览器缓存

### 2.4 sale_type 映射到数据库常量

在 [Constants.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Config/Constants.php#L136-L147) 定义：

| sale_type 参数 | sale_status 常量 | sale_type 常量 |
|---------------|-----------------|----------------|
| `complete` | COMPLETED (0) | POS(0) + INVOICE(1) + RETURN(4) |
| `sales` | COMPLETED (0) | POS(0) + INVOICE(1) |
| `quotes` | SUSPENDED (1) | QUOTE(3) |
| `work_orders` | SUSPENDED (1) | WORK_ORDER(2) |
| `canceled` | CANCELED (2) | 不限 |
| `returns` | COMPLETED (0) | RETURN(4) |

---

## 三、查询构造机制详解

报表查询构造分为**两大模式**：

### 3.1 模式一：Summary_report 模板方法模式

适用报表：`Summary_sales`、`Summary_categories`、`Summary_items`、`Summary_customers`、`Summary_suppliers`、`Summary_employees`

核心在 [Summary_report.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_report.php)。

#### 3.1.1 执行流程（模板方法）

```
getData($inputs)  [Summary_report 基类]
  ├── 1. 创建临时表 sales_items_taxes_temp  (按日期范围预汇总税额)
  ├── 2. 创建临时表 sales_payments_temp     (按日期范围预汇总支付)
  ├── 3. $builder = db->table('sales_items')
  ├── 4. _select($inputs, $builder)         ← 子类扩展字段
  │     └── parent::_select() → __common_select()  → 计算 subtotal/tax/total/cost/profit
  ├── 5. _from($builder)                    ← 子类扩展 JOIN
  │     └── parent::_from() → __common_from()  → JOIN sales / 临时表
  ├── 6. _where($inputs, $builder)          ← 子类扩展条件
  │     └── parent::_where() → __common_where() → 日期范围 + location + sale_type
  ├── 7. _group_order($builder)             ← 子类定义分组排序
  └── 8. 返回 getResultArray()
```

#### 3.1.2 公共 SQL 构造逻辑 (`__common_select`)

关键计算公式（[Summary_report.php#L13-L87](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_report.php#L13-L87)）：

```php
// 折扣后销售单价（根据折扣类型 PERCENT/FIXED 分支）
$sale_price = 'CASE WHEN sales_items.discount_type = ' . PERCENT
    . " THEN quantity * unit_price - ROUND(quantity * unit_price * discount/100, $decimals) "
    . 'ELSE quantity * (unit_price - discount) END';

// 含税/不含税模式切换
if ($config['tax_included']) {
    $sale_total    = "ROUND(SUM($sale_price), $decimals) + $cash_adjustment";
    $sale_subtotal = "$sale_total - $sales_tax";
} else {
    $sale_subtotal = "ROUND(SUM($sale_price), $decimals) + $cash_adjustment";
    $sale_total    = "ROUND(SUM($sale_price), $decimals) + $sales_tax + $cash_adjustment";
}
```

#### 3.1.3 公共 WHERE 条件 (`__common_where`)

```php
// 日期格式：纯日期 vs 日期时间
if (empty($config['date_or_time_format'])) {
    $builder->where('DATE(sales.sale_time) BETWEEN ...');
} else {
    $builder->where('sales.sale_time BETWEEN ...');   // 带 rawurldecode 解码
}

// 库存位置过滤
if ($inputs['location_id'] != 'all') {
    $builder->where('sales_items.item_location', $inputs['location_id']);
}

// sale_type 分支（参见 2.4 映射表）
```

#### 3.1.4 子类扩展差异示例

| 报表 | _select 扩展 | _from 扩展 JOIN | _group_order |
|------|-------------|-----------------|-------------|
| [Summary_sales](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_sales.php) | DATE(sale_time) + SUM(quantity) + COUNT(DISTINCT sale_id) | 无 | GROUP BY sale_date |
| [Summary_items](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_items.php) | MAX(name/category/price) + SUM(quantity) | JOIN items | GROUP BY items.item_id |
| [Summary_categories](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_categories.php) | items.category + SUM(quantity) | JOIN items | GROUP BY category |
| [Summary_customers](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_customers.php) | MAX(客户姓名) + SUM(quantity) + COUNT(sale_id) | JOIN people AS customer_p | GROUP BY sales.customer_id |
| [Summary_employees](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_employees.php) | MAX(员工姓名) + SUM(quantity) + COUNT(sale_id) | JOIN people AS employee_p | GROUP BY sales.employee_id |
| [Summary_suppliers](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_suppliers.php) | MAX(供应商信息) + SUM(quantity) | JOIN items + suppliers + people | GROUP BY items.supplier_id |

#### 3.1.5 getSummaryData 的构造

汇总数据同样使用 `__common_select` + `__common_from` + `_where`，但**不调用子类的 `_group_order`**，直接返回一行总计。

---

### 3.2 模式二：详细报表的临时表模式

适用报表：`Detailed_sales`、`Detailed_receivings`、`Specific_customer`、`Specific_employee`、`Specific_discount`、`Specific_supplier`

#### 3.2.1 核心机制：预生成 sales_items_temp 宽表

所有详细报表首先调用 `Sale::create_temp_table($inputs)` 创建三张临时表：

**调用链**：
```
Detailed_sales::create($inputs)
  └── Sale::create_temp_table($inputs)
        ├── 1. CREATE TEMP TABLE sales_items_taxes_temp  (MEMORY引擎)
        ├── 2. CREATE TEMP TABLE sales_payments_temp
        └── 3. CREATE TEMP TABLE sales_items_temp       ← 核心宽表
```

#### 3.2.2 sales_items_temp 宽表结构

在 [Sale.php#L1106-L1172](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Sale.php#L1106-L1172) 创建，包含 30+ 字段的汇总宽表：

| 字段来源 | 关联表 | 典型字段 |
|---------|--------|---------|
| 销售主表 | sales | sale_id, sale_time, sale_status, sale_type, comment, customer_id, employee_id |
| 客户信息 | people + customers | customer_name, customer_email, customer_company_name |
| 员工信息 | people | employee_name |
| 商品信息 | items | item_id, name, item_number, category, supplier_id, cost_price, unit_price |
| 销售明细 | sales_items | quantity_purchased, discount, discount_type, serialnumber, item_location, description |
| 支付信息 | sales_payments_temp | payment_type, sale_payment_amount |
| 税额计算 | sales_items_taxes_temp | subtotal, tax, total |
| 成本利润 | 计算字段 | cost, profit |

**索引**：`(sale_date)`、`(sale_time)`、`(sale_id)`

**临时表引擎**：`ENGINE=MEMORY`（税表）/ 默认 InnoDB

#### 3.2.3 详细报表查询流程

以 [Detailed_sales::getData()](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Detailed_sales.php#L96-L210) 为例：

```php
public function getData(array $inputs): array
{
    // 第一步：查询销售摘要列表（每 sale_id 一行）
    $builder = db->table('sales_items_temp');
    $builder->select('sale_id, MAX(sale_time), SUM(subtotal), ...');
    $builder->where(/* location_id + sale_type 条件 */);
    $builder->groupBy('sale_id');
    $data['summary'] = $builder->get()->getResultArray();

    // 第二步：N+1 查询每行销售的商品明细
    foreach ($data['summary'] as $key => $value) {
        $builder = db->table('sales_items_temp');
        $builder->select('name, category, quantity, subtotal, tax, ...');
        // 如果有自定义属性还要 JOIN attribute_links + attribute_values
        $builder->where('sale_id', $value['sale_id']);
        $builder->groupBy('sale_id, item_id, sale_id');
        $data['details'][$key] = $builder->get()->getResultArray();

        // 第三步：每个 sale_id 再查积分记录
        $builder = db->table('sales_reward_points');
        $builder->where('sale_id', $value['sale_id']);
        $data['rewards'][$key] = $builder->get()->getResultArray();
    }

    return $data;
}
```

**⚠️ 性能热点**：N+1 查询问题 —— 如果有 M 笔销售，会执行 `1 + 2*M` 次查询。

---

### 3.3 模式三：完全自定义查询

以下报表完全重写了 `getData()` / `getSummaryData()`，不使用基类模板方法：

| 报表 | 原因 | 特殊实现 |
|------|------|---------|
| [Summary_taxes](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_taxes.php) | 税额按行计算而非按商品聚合 | 使用 `fromSubquery()` 子查询模式 |
| [Summary_sales_taxes](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_sales_taxes.php) | 数据源为 sales_taxes 表（非 sales_items） | 直接 JOIN tax_categories + tax_jurisdictions |
| [Summary_discounts](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_discounts.php) | 需区分 PERCENT/FIXED 两种折扣类型显示 | 动态拼接 SELECT + 按折扣值 GROUP BY |
| [Summary_payments](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_payments.php) | 需生成销售类型与支付方式两段数据，中间加分隔行 | 创建 3 张专用临时表，两段 UNION 风格查询 + PHP 层合并数组 |
| [Summary_expenses_categories](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_expenses_categories.php) | 非销售类数据，数据源为 expenses 表 | 直接查询 expenses + expense_categories |
| [Inventory_summary](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Inventory_summary.php) | 当前库存快照，无时间范围 | 查询 items + item_quantities + stock_locations |
| [Inventory_low](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Inventory_low.php) | 库存告警条件 | quantity <= reorder_level |

---

## 四、模型选择逻辑与路由对应关系

### 4.1 控制器 → 模型映射表

| 控制器方法 | 实例化的模型 | 视图模板 |
|-----------|-------------|---------|
| `summary_sales()` | `$this->summary_sales` (Summary_sales) | reports/tabular |
| `summary_categories()` | `$this->summary_categories` (Summary_categories) | reports/tabular |
| `summary_items()` | `$this->summary_items` (Summary_items) | reports/tabular |
| `summary_customers()` | `$this->summary_customers` (Summary_customers) | reports/tabular |
| `summary_suppliers()` | `$this->summary_suppliers` (Summary_suppliers) | reports/tabular |
| `summary_employees()` | `$this->summary_employees` (Summary_employees) | reports/tabular |
| `summary_taxes()` | `$this->summary_taxes` (Summary_taxes) | reports/tabular |
| `summary_sales_taxes()` | `$this->summary_sales_taxes` (Summary_sales_taxes) | reports/tabular |
| `summary_discounts()` | `$this->summary_discounts` (Summary_discounts) | reports/tabular |
| `summary_payments()` | `$this->summary_payments` (Summary_payments) | reports/tabular |
| `summary_expenses_categories()` | `$this->summary_expenses_categories` (Summary_expenses_categories) | reports/tabular |
| `detailed_sales()` | `$this->detailed_sales` (Detailed_sales) | reports/tabular_details |
| `detailed_receivings()` | `$this->detailed_receivings` (Detailed_receivings) | reports/tabular_details |
| `specific_customers()` | 动态 `model(Specific_customer::class)` | reports/tabular_details |
| `specific_employees()` | 动态 `model(Specific_employee::class)` | reports/tabular_details |
| `specific_discounts()` | 动态 `model(Specific_discount::class)` | reports/tabular_details |
| `specific_suppliers()` | 动态 `model(Specific_supplier::class)` | reports/tabular |
| `inventory_summary()` | `$this->inventory_summary` (Inventory_summary) | reports/tabular |
| `inventory_low()` | 动态 `model(Inventory_low::class)` | reports/tabular |
| `graphical_*()` | 对应 summary 模型 | reports/graphical (内含 pie/line/bar/hbar 子视图) |

### 4.2 图形报表与表格报表的关系

图形报表（`graphical_summary_xxx`）**复用完全相同的模型和 `getData()`**，只是控制器层对结果做了一次转换：

```php
// graphical_summary_sales() 示例
$report_data = $this->summary_sales->getData($inputs);
$summary     = $this->summary_sales->getSummaryData($inputs);

$labels = [];
$series = [];
foreach ($report_data as $row) {
    $labels[] = to_date(strtotime($row['sale_date']));
    $series[] = ['meta' => $date, 'value' => $row['total']];
}

return view('reports/graphical', [
    'chart_type'    => 'reports/graphs/line',   // pie / hbar / bar / line
    'labels_1'      => $labels,
    'series_data_1' => $series,
    // ...
]);
```

---

## 五、大范围统计的性能影响分析

### 5.1 性能开销矩阵

| 报表类型 | 查询次数 | 临时表 | 全表扫描风险 | JOIN 数量 | 数据量膨胀 |
|---------|---------|-------|------------|----------|----------|
| Summary_sales 等（模板方法） | **3** (2 建临时表 + 1 查询) | 2 张 MEMORY 临时表 | 高（日期范围扫描 sales_items） | 主查询 4-6 张表 | 低 |
| Detailed_sales | **1 + 2N** (N=sale 数量) | 3 张临时表 | 高（sales_items 全量宽表） | 首次 9+ 张表 | 极高（按行 item 展开） |
| Specific_xxx | **1 + 2N** | 3 张临时表 | 中（有 customer_id/employee_id 过滤） | 同 Detailed_sales | 中 |
| Summary_payments | **5** (3 建临时表 + 2 段查询) | 3 张 MEMORY 临时表 | 高 | 3-4 张表 | 中 |
| Summary_taxes | **1** (子查询) | 无 | 高 | 3 张表 | 低 |
| Inventory_summary | **1** | 无 | 中（items 全表） | 3 张表 | 取决于 SKU 数量 |

### 5.2 关键性能瓶颈

#### 瓶颈 1：临时表构建（大范围日期）

`Sale::create_temp_table()` 中的三条 CREATE TEMP TABLE 语句都会扫描大量数据：

```sql
-- 扫描 sales + sales_items + sales_items_taxes 三表关联
CREATE TEMPORARY TABLE sales_items_taxes_temp ENGINE=MEMORY
SELECT ... FROM sales_items_taxes 
INNER JOIN sales ON ... INNER JOIN sales_items ON ...
WHERE DATE(sale_time) BETWEEN ? AND ? 
GROUP BY sale_id, item_id, line;
```

- 如果查询**一年数据**，可能涉及百万行级别的 `sales_items` 聚合
- MEMORY 引擎临时表有大小限制（`max_heap_table_size`），超限时会转磁盘
- 临时表**不会跨请求复用**，每次报表请求都会重建

#### 瓶颈 2：日期列上的函数运算

```php
// 不使用索引的写法
$builder->where('DATE(sales.sale_time) BETWEEN ...');
```

`DATE(sale_time)` 会导致 `sale_time` 索引失效，退化为全表扫描。仅当配置 `date_or_time_format` 非空时才走范围扫描。

#### 瓶颈 3：详细报表的 N+1 查询

`Detailed_sales::getData()` 中：
```php
foreach ($data['summary'] as $key => $value) {
    // 每个 sale_id 执行 2 次查询（明细 + 积分）
    $data['details'][$key] = /* 查询明细 */;
    $data['rewards'][$key] = /* 查询积分 */;
}
```

大范围日期下（如 10000 笔销售），将产生 **20001 次 SQL 查询**，严重影响响应时间。

#### 瓶颈 4：深度 JOIN

`sales_items_temp` 创建时 JOIN 了 **7-9 张表**：

```
sales_items 
  → sales 
  → items 
  → sales_payments_temp（本身也是聚合结果）
  → suppliers 
  → people (customer) 
  → customers 
  → people (employee)
  → sales_items_taxes_temp（本身也是聚合结果）
```

这使得临时表构建本身就是一个重查询。

#### 瓶颈 5：Summary_payments 的多次临时表 + UPDATE

[Summary_payments.php#L135-L190](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Models/Reports/Summary_payments.php#L135-L190) 创建 3 张临时表后还执行了一条：

```sql
UPDATE sumpay_items_temp SET trans_amount = trans_amount + 
    IFNULL((SELECT total_taxes FROM sumpay_taxes_temp WHERE ...), 0)
```

这是**逐行相关子查询更新**，临时表大时性能极差。

### 5.3 已有的性能优化手段

| 优化点 | 位置 | 效果 |
|-------|------|------|
| MEMORY 引擎临时表 | 税表创建时指定 `ENGINE=MEMORY` | 内存聚合，减少磁盘 IO |
| 临时表索引 | `(sale_id)`, `(item_id)`, `(sale_date)`, `(sale_time)` | 加速后续 JOIN 和 WHERE |
| 预聚合到宽表 | `sales_items_temp` 一次性算好 subtotal/tax/total/cost/profit | 避免后续重复计算折扣/税额 |
| 控制器响应头禁用缓存 | `clearCache()` 设置 no-cache 头 | 防止浏览器缓存过期数据 |

### 5.4 潜在优化建议

1. **为 `sale_time` 添加独立日期列**或使用生成列，避免 `DATE()` 函数导致索引失效
2. **详细报表 N+1 改写**：用 `WHERE sale_id IN (...)` 批量查询明细和积分，PHP 层按 sale_id 分组
3. **临时表复用**：同一会话内相同参数的报表请求可复用缓存（如使用 Redis 或查询结果缓存）
4. **Summary_payments 的 UPDATE 改 JOIN**：用 UPDATE...JOIN 替代相关子查询
5. **分页支持**：当前所有报表都是一次性加载全部数据，大范围日期下应支持分页

---

## 六、数据流全景图

```
┌───────────────────────────────────────────────────────────────────────┐
│  用户访问 reports/summary_items                                         │
│      ↓                                                                  │
│  Routes.php: 匹配 summary_(:any) → Reports::date_input()               │
│      ↓                                                                  │
│  渲染 date_input.php 表单，用户选择日期/销售类型/仓库                    │
│      ↓                                                                  │
│  JS 拼接 URL: /reports/summary_items/2024-01-01/2024-12-31/complete/all│
│      ↓                                                                  │
│  Reports::summary_items($start, $end, $sale_type, $location_id)         │
│      ├─ 组装 $inputs = [start_date, end_date, sale_type, location_id]   │
│      ├─ Summary_items::getData($inputs)                                 │
│      │     ├─ Summary_report::__common_select() 建 2 张临时表           │
│      │     ├─ Summary_report::getData() 模板方法                        │
│      │     │     ├─ _select()  → 加 name,category,quantity 字段        │
│      │     │     ├─ _from()    → JOIN items 表                         │
│      │     │     ├─ _where()   → 日期 + location + sale_type           │
│      │     │     └─ _group_order() → GROUP BY items.item_id            │
│      │     └─ 返回按商品聚合的结果数组                                   │
│      ├─ Summary_items::getSummaryData($inputs)  → 总计行               │
│      ├─ 控制器层 PHP 格式化（to_currency / to_quantity_decimals）       │
│      └─ 传数据给 views/reports/tabular.php 渲染表格                     │
└───────────────────────────────────────────────────────────────────────┘
```

对于详细报表流程：

```
┌───────────────────────────────────────────────────────────────────────┐
│  Reports::detailed_sales($start, $end, $sale_type, $location_id)        │
│      ├─ $inputs = [..., definition_ids] （自定义属性字段）              │
│      ├─ Detailed_sales::create($inputs)                                 │
│      │     └─ Sale::create_temp_table($inputs)                          │
│      │           ├─ CREATE TEMP sales_items_taxes_temp                  │
│      │           ├─ CREATE TEMP sales_payments_temp                     │
│      │           └─ CREATE TEMP sales_items_temp (30+字段宽表)          │
│      ├─ Detailed_sales::getData($inputs)                                │
│      │     ├─ SELECT ... FROM sales_items_temp GROUP BY sale_id         │
│      │     └─ foreach (每笔 sale_id):                                   │
│      │           ├─ SELECT 商品明细 FROM sales_items_temp               │
│      │           │     (有自定义属性时 JOIN attribute_links/values)     │
│      │           └─ SELECT 积分 FROM sales_reward_points                │
│      └─ views/reports/tabular_details.php 渲染（支持展开明细行）         │
└───────────────────────────────────────────────────────────────────────┘
```
