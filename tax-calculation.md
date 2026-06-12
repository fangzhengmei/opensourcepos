# OSPOS 税费计算机制深度解析

## 1. 核心架构概览

OSPOS 的税费计算系统由两套互斥的计算模式构成，通过全局配置 `use_destination_based_tax` 切换：

| 模式 | 配置值 | 税率数据源 | 适用场景 |
|------|--------|-----------|---------|
| 基础税制（Base System） | `false` | `items_taxes` 表（商品直接绑定税率） | 简单单店铺、税制单一的场景 |
| 目的地税制（Destination-based） | `true` | `tax_rates` 表（税码 + 税类 + 辖区三维组合） | 多地区经营、存在多级税区的场景 |

核心入口：[Tax_lib::get_taxes()](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Libraries/Tax_lib.php#L86-L171)

---

## 2. 数据模型与实体关系

### 2.1 实体关系图

```
商品 (items)
  ├─ tax_category_id ──────┐
  │                        ▼
  │              税类 (tax_categories)
  │                - tax_category_id (PK)
  │                - tax_category (名称)
  │                - tax_group_sequence (组内排序)
  │
  │  基础税制:
  └─── items_taxes (item_id, name, percent)
           每个商品可绑定多条独立税率

  目的地税制:
  税码 (tax_codes)         辖区 (tax_jurisdictions)
    - tax_code_id (PK)       - jurisdiction_id (PK)
    - city, state            - tax_group (税种分组名)
                             - tax_type (价内/价外)
                             - tax_group_sequence (排序)
                             - cascade_sequence (级次)
         │                       │
         └───────────┬───────────┘
                     ▼
              税率 (tax_rates)
                - rate_tax_code_id (FK)
                - rate_tax_category_id (FK)
                - rate_jurisdiction_id (FK)
                - tax_rate (%)
                - tax_rounding_code
```

### 2.2 关键模型文件

- [Tax.php](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Models/Tax.php) — `tax_rates` 表模型，负责目的地税制下的税率查询
- [Tax_category.php](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Models/Tax_category.php) — `tax_categories` 表模型
- [Tax_code.php](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Models/Tax_code.php) — `tax_codes` 表模型
- [Tax_jurisdiction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Models/Tax_jurisdiction.php) — `tax_jurisdictions` 表模型
- [Item_taxes.php](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Models/Item_taxes.php) — `items_taxes` 表模型（基础税制）

---

## 3. 税类（Tax Category）

### 3.1 定义

税类是对商品进行税务分类的维度，例如："标准税率"、"减免税率"、"零税率"。

表结构定义见 [Tax_category::$allowedFields](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Models/Tax_category.php#L19-L23)：

| 字段 | 说明 |
|------|------|
| `tax_category_id` | 主键 |
| `tax_category` | 税类名称 |
| `tax_group_sequence` | 税种分组显示顺序（与辖区的同名字段相加决定最终打印顺序） |
| `deleted` | 软删除标记 |

### 3.2 税类与商品的关联

在 `items` 表中通过 `tax_category_id` 字段关联，见 [Item.php](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Models/Item.php#L52)。

**优先级规则（目的地税制）**：
1. 若商品自身的 `tax_category_id` 不为空 → 使用商品税类
2. 否则使用系统全局配置 `default_tax_category`

代码位置：[Tax_lib::get_taxes()](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Libraries/Tax_lib.php#L150-L152)：

```php
if ($item['tax_category_id'] == null) {
    $item['tax_category_id'] = $this->config['default_tax_category'];
}
```

商品表单中的税类配置（仅目的地税制下可见）见 [items/form.php](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Views/items/form.php#L267-L279)。

---

## 4. 商品税率配置

### 4.1 基础税制（`use_destination_based_tax = false`）

每个商品直接在 `items_taxes` 表中配置多条独立税率：

| 字段 | 说明 |
|------|------|
| `item_id` | 商品 ID |
| `name` | 税种名称（如"增值税"、"消费税"） |
| `percent` | 税率百分比（如 13.00） |

商品保存逻辑见 [Items.php](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Controllers/Items.php#L694-L708)：

```php
$tax_names = $this->request->getPost('tax_names');
$tax_percents = $this->request->getPost('tax_percents');
foreach ($tax_percents as $tax_percent) {
    $tax_percentage = parse_tax($tax_percent);
    if (is_numeric($tax_percentage)) {
        $items_taxes_data[] = ['name' => $tax_names[$tax_name_index], 'percent' => $tax_percentage];
    }
    $tax_name_index++;
}
$success &= $this->item_taxes->save_value($items_taxes_data, $item_id);
```

表单界面中，默认值来源：
- 税种名称：`$config['default_tax_1_name']` / `$config['default_tax_2_name']`
- 税率值：`$config['default_tax_1_rate']` / `$config['default_tax_2_rate']`

### 4.2 目的地税制（`use_destination_based_tax = true`）

税率通过 `tax_rates` 表定义，由三个维度联合确定：
- **税码**（`rate_tax_code_id`）：代表销售发生地的税务编码
- **税类**（`rate_tax_category_id`）：商品的税务分类
- **辖区**（`rate_jurisdiction_id`）：税务管辖区（国家/州/市等）

税率查询见 [Tax::get_taxes()](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Models/Tax.php#L117-L131)：

```sql
WHERE rate_tax_code_id = ? AND rate_tax_category_id = ?
ORDER BY cascade_sequence, tax_group, jurisdiction_name, 
         tax_jurisdictions.tax_group_sequence + tax_categories.tax_group_sequence
```

每个辖区还可以配置：
- `tax_type`：价内税（VAT，`'0'`）或价外税（Sales Tax，`'1'`）
- `cascade_sequence`：级次（用于税上加税的复合计税）
- `tax_group_sequence`：打印顺序

---

## 5. 税率覆盖规则与优先级

### 5.1 税制模式切换优先级（最高层）

配置项 `use_destination_based_tax` 是**最高优先级开关**，决定了使用哪套计算逻辑，两者完全互斥。

在配置界面 [tax_config.php](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Views/configs/tax_config.php#L146-L156) 中可见：
- 勾选目的地税制时：基础税制的 `tax_included`、`default_tax_1/2_rate`、`default_tax_1/2_name` 全部禁用
- 不勾选时：目的地税制的 `default_tax_code`、`default_tax_category`、`default_tax_jurisdiction` 全部禁用

### 5.2 税码（Tax Code）匹配优先级（目的地税制）

税码决定了适用哪套地区税率，见 [Tax_lib::get_applicable_tax_code()](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Libraries/Tax_lib.php#L373-L388)：

```
优先级 1：收银模式为 'sale' 时
   → 直接使用 config['default_tax_code']（强制覆盖客户设置）

优先级 2：客户已指定 sales_tax_code_id（非 0 非 null）
   → 使用客户的 sales_tax_code_id

优先级 3：按客户所在城市+州精确匹配
   → Tax_code::get_sales_tax_code(city, state)

优先级 4：按客户所在州模糊匹配（city=''）
   → get_sales_tax_code('', state)

优先级 5：回退到系统默认
   → config['default_tax_code']
```

### 5.3 税类（Tax Category）优先级（目的地税制）

见 [Tax_lib::get_taxes()](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Libraries/Tax_lib.php#L150-L152)：

```
优先级 1：商品自身 items.tax_category_id 非空
   → 使用商品税类

优先级 2：回退到系统默认
   → config['default_tax_category']
```

### 5.4 客户应税资格判定

两套税制通用，见 [Tax_lib::get_taxes()](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Libraries/Tax_lib.php#L96)：

```
客户未选择（customer_id == -1，散客）→ 计税
客户已选择且 customer.taxable == 1   → 计税
客户已选择且 customer.taxable == 0   → 不计税
```

### 5.5 完整优先级链

```
是否计税？
  ├─ 客户未选 OR customer.taxable = 1 → 继续
  └─ 否则 → 跳过，税额为 0

use_destination_based_tax ?
  ├─ 是（目的地税制）
  │    ├─ 确定税码：sale_mode_default > customer_code > city+state > state > default_code
  │    ├─ 确定税类：item.tax_category_id > default_tax_category
  │    └─ 查询税率：tax_rates WHERE tax_code + tax_category
  │         ├─ 按 cascade_sequence（级次）排序
  │         └─ 同级内按 tax_group + jurisdiction + 组合顺序排序
  │
  └─ 否（基础税制）
       └─ 直接取 items_taxes WHERE item_id（按存储顺序，最多 2 条：税1、税2）
```

---

## 6. 计税基础（Tax Basis）计算

### 6.1 税基公式

税基 = 商品单价 × 数量 − 折扣金额

实现见 [Sale_lib::get_item_total()](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Libraries/Sale_lib.php#L1579-L1589)：

```php
public function get_item_total(string $quantity, string $price, string $discount, int $discount_type, bool $include_discount = false): string
{
    $total = bcmul($quantity, $price);
    if ($include_discount) {
        $discount_amount = $this->get_item_discount($quantity, $price, $discount, $discount_type);
        return bcsub($total, $discount_amount);
    }
    return $total;
}
```

**关键点**：`$include_discount = true` 时返回折扣后的金额作为税基。

### 6.2 价外税（Sales Tax / Excluded）

税额 = 税基 × (税率 / 100)

实现见 [Tax_lib::get_tax_for_amount()](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Libraries/Tax_lib.php#L76-L81)：

```php
public function get_tax_for_amount(string $tax_basis, string $tax_percentage, int $rounding_mode, int $decimals): string
{
    $tax_amount = bcmul($tax_basis, bcdiv($tax_percentage, '100'));
    return rounding_mode::round_number($rounding_mode, $tax_amount, $decimals);
}
```

### 6.3 价内税（VAT / Included）

从含税总价中反推税额：

- 不含税价格 = 含税总价 / (1 + 税率/100)
- 税额 = 含税总价 − 不含税价格

实现见 [Tax_lib::get_included_tax()](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Libraries/Tax_lib.php#L183-L189)：

```php
public function get_included_tax(string $quantity, string $price, string $discount_percentage, int $discount_type, string $tax_percentage, $tax_decimal, $rounding_code): string
{
    $item_total = $this->sale_lib->get_item_total($quantity, $price, $discount_percentage, $discount_type, true);
    $tax_fraction = bcdiv(bcadd('100', $tax_percentage), '100');
    $price_tax_excl = bcdiv($item_total, $tax_fraction);
    return bcsub($item_total, $price_tax_excl);
}
```

> **注意**：`$tax_decimal` 和 `$rounding_code` 参数在此函数中未使用，价内税计算结果**不做单项舍入**，而是等到最后统一舍入。

### 6.4 级次计税（Cascading Tax）

仅目的地税制支持，见 [Tax_lib::apply_destination_tax()](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Libraries/Tax_lib.php#L317-L338)：

```php
$last_cascade_sequence = 0;
$cascade_tax_amount = '0.0';

foreach ($tax_definition as $tax) {
    $cascade_sequence = $tax['cascade_sequence'];
    if ($cascade_sequence != $last_cascade_sequence) {
        $last_cascade_sequence = $cascade_sequence;
        $tax_basis = bcadd($tax_basis, $cascade_tax_amount);  // 税上加税
    }
    // ... 计算当前级次税额
    $cascade_tax_amount = bcadd($cascade_tax_amount, $tax_amount);
}
```

规则：
- 同一 `cascade_sequence` 的税种在同一税基上计算
- 进入下一级次时，将之前所有级次的税额累加进税基
- 价内税（`tax_type = '0'`）不参与级次累加

---

## 7. 舍入边界与舍入模式

### 7.1 舍入精度选择

最终舍入时使用的小数位由全局配置 `tax_included` 决定，见 [Tax_lib::round_taxes()](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Libraries/Tax_lib.php#L263-L267)：

| 配置 | 小数位来源 | 配置项 |
|------|-----------|--------|
| `tax_included = true`（价内税） | `tax_decimals()` | `config['tax_decimals']` |
| `tax_included = false`（价外税） | `totals_decimals()` | `config['currency_decimals']` |

函数定义见 [locale_helper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Helpers/locale_helper.php#L309-L331)。

### 7.2 七种舍入模式

定义见 [Rounding_mode.php](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Models/Enums/Rounding_mode.php#L9-L15)：

| 常量 | 值 | PHP 对应 | 说明 |
|------|---|---------|------|
| `HALF_UP` | 1 | `PHP_ROUND_HALF_UP` | 四舍五入（0.5 进 1） |
| `HALF_DOWN` | 2 | `PHP_ROUND_HALF_DOWN` | 五舍六入（0.5 舍去） |
| `HALF_EVEN` | 3 | `PHP_ROUND_HALF_EVEN` | 银行家舍入（0.5 向偶数靠） |
| `HALF_ODD` | 4 | `PHP_ROUND_HALF_ODD` | 0.5 向奇数靠 |
| `ROUND_UP` | 5 | — | 向上进位（只要有尾数就进 1） |
| `ROUND_DOWN` | 6 | — | 向下截断（无论尾数多少都舍去） |
| `HALF_FIVE` | 7 | — | 舍入到最近的 5 的倍数 |

实现见 [Rounding_mode::round_number()](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Models/Enums/Rounding_mode.php#L70-L85) 和 [Tax_lib::round_taxes()](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Libraries/Tax_lib.php#L252-L294)。

### 7.3 舍入时机与边界

税费计算采用 **「分项精确累加 + 最终统一舍入」** 的整体策略，但**价内税和价外税的单项舍入行为完全不同**。

#### 7.3.1 BC Math 全局精度

所有 `bc*` 函数的运算精度由全局设置决定，见 [Load_config.php](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Events/Load_config.php#L44)：

```php
bcscale(max(2, totals_decimals() + tax_decimals()));
```

这意味着所有高精度运算（`bcmul`、`bcdiv`、`bcadd`、`bcsub`）默认使用 `max(2, currency_decimals + tax_decimals)` 位小数，通常为 4 位。

#### 7.3.2 完整舍入流程

```
阶段 0：全局精度设置
  └─ bcscale(max(2, totals_decimals() + tax_decimals())) — 默认 4 位小数

阶段 1：单项税额计算（价内/价外差异最大）
  ├─ 价外税（Excluded Tax）：
  │    计算：bcmul(tax_basis, bcdiv(tax_rate, 100))
  │    舍入：调用 round_number(rounding_mode, tax_amount, tax_decimals)
  │          按 tax_decimals 做单项舍入，触发 HALF_ODD/HALF_FIVE 逻辑
  │
  └─ 价内税（Included Tax）：
       计算：item_total - (item_total / (1 + tax_rate/100))
       舍入：无单项舍入！get_included_tax() 的 $tax_decimal 和 $rounding_code 参数未使用
             结果为 bc 运算的高精度值（通常 4 位小数）

阶段 2：按税种分组累加（4 位小数精度）
  └─ update_taxes() 中 bcadd(..., ..., 4)，见 Tax_lib.php#L223-L224
     $taxes[$group]['sale_tax_basis'] = bcadd($old_basis, $new_basis, 4);
     $taxes[$group]['sale_tax_amount'] = bcadd($old_amount, $new_amount, 4);

阶段 3：最终统一舍入（按税种分组）
  └─ round_taxes() 中对每个税种分组的总税额做最终舍入
     精度：由全局配置 tax_included 决定
           tax_included = true  → 使用 tax_decimals
           tax_included = false → 使用 currency_decimals
     模式：使用该税种的 rounding_code，但触发 round_taxes() 中的另一套实现
```

**关键差异总结**：
- **价外税**：经历两次舍入（单项舍入 + 最终舍入），单项舍入会触发 HALF_ODD/HALF_FIVE 逻辑
- **价内税**：只经历一次舍入（仅最终舍入），单项阶段完全不做舍入
- 同一税种的所有商品税额会先求和、后舍入（而非每个商品先舍入、后求和）

### 7.4 舍入边界示例

假设 `currency_decimals = 2`，模式为 `HALF_UP`：

| 税前总额 | 税率 | 精确税额 | 舍入后 |
|---------|------|---------|-------|
| 10.00 | 13% | 1.3000 | 1.30 |
| 10.01 | 13% | 1.3013 | 1.30 |
| 10.04 | 13% | 1.3052 | 1.31 |
| 99.99 | 13% | 12.9987 | 13.00 |

### 7.5 HALF_ODD：两处实现的不一致性

#### 7.5.1 为什么会有两处实现

税费计算中存在**两套独立的舍入实现**，分别服务于不同的计算阶段：

| 阶段 | 调用路径 | 实现函数 | 适用场景 |
|------|---------|---------|---------|
| 单项税额计算 | `get_tax_for_amount()` | [Rounding_mode::round_number()](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Models/Enums/Rounding_mode.php#L70-L85) | 目的地税制下，每个商品每个税种的单项税额计算 |
| 最终汇总舍入 | `round_taxes()` | [Tax_lib::round_taxes()](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Libraries/Tax_lib.php#L252-L294) | 按税种分组累加后，对总税额做最终舍入 |

> **注意**：基础税制下两处都硬编码为 `HALF_UP`，因此不存在不一致问题。仅**目的地税制**下会触发 `tax_rates.tax_rounding_code` 配置的其他舍入模式。

#### 7.5.2 HALF_ODD 的两处实现对比

**单项计算 — Rounding_mode::round_number()**：

```php
} else {
    $rounded_total = round($amount, $decimals, $rounding_mode);
}
```

此处 `$rounding_mode` 直接使用传入的 `Rounding_mode::HALF_ODD`（值为 4），即 PHP 原生的 `PHP_ROUND_HALF_ODD`。**行为正确**：当小数部分恰好为 0.5 时，向最近的奇数舍入。

**最终汇总 — Tax_lib::round_taxes()**：

```php
} elseif ($rounding_code == Rounding_mode::HALF_ODD) {
    $rounded_tax_amount = round($tax_amount, $decimals, PHP_ROUND_HALF_UP);
}
```

此处**硬编码为 `PHP_ROUND_HALF_UP`**，而非 `PHP_ROUND_HALF_ODD`。这是一个实现错误：当舍入模式为 HALF_ODD 时，最终汇总阶段实际执行的是 HALF_UP（四舍五入）。

#### 7.5.3 数值差异示例

设 `decimals = 2`，待舍入值恰好为 `x.xx5`（即 0.5 边界）：

| 待舍入值 | 正确 HALF_ODD 结果 | 实际（HALF_UP）结果 | 差异 |
|---------|-------------------|-------------------|------|
| 10.115 | 10.11（第 2 位小数 1 是奇数，舍去 5） | 10.12（四舍五入，5 进 1） | 差 0.01 |
| 10.125 | 10.13（第 2 位小数 2 是偶数，向奇数 3 靠拢） | 10.13（四舍五入，5 进 1） | 相同 |
| 10.135 | 10.13（第 2 位小数 3 是奇数，舍去 5） | 10.14（四舍五入，5 进 1） | 差 0.01 |

**结论**：当小数点第 `decimals+1` 位恰好为 5 且第 `decimals` 位是奇数时，两处实现会产生 `1` 个最低位的差异。

### 7.6 HALF_FIVE：两处实现的三处不一致

#### 7.6.1 三处实现代码对比

代码库中实际上存在 **三套** HALF_FIVE 实现，分别用于不同场景：

| 位置 | 函数 | 代码 |
|------|------|------|
| 单项计算 | [Rounding_mode::round_number()](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Models/Enums/Rounding_mode.php#L78-L79) | `round($amount / 5, $decimals, Rounding_mode::HALF_EVEN) * 5` |
| 最终汇总 | [Tax_lib::round_taxes()](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Libraries/Tax_lib.php#L288-L289) | `round($tax_amount / 5) * 5` |
| 数据迁移 | [Migration_Sales_Tax_Data::round_number()](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Database/Migrations/20170502221506_sales_tax_data.php#L278-L279) | `round($amount / 5) * 5` |

迁移文件的实现与 `Tax_lib::round_taxes()` 一致。

#### 7.6.2 差异一：小数位参数

**单项计算**：`round($amount / 5, $decimals, ...)` —— 明确传入了 `$decimals` 参数
**最终汇总**：`round($tax_amount / 5)` —— **未传入 precision 参数**，PHP `round()` 默认 precision 为 `0`

这意味着：
- 单项计算：先除以 5，保留 `$decimals` 位小数，再乘以 5
- 最终汇总：先除以 5，**保留 0 位小数**（即整数），再乘以 5

**数值差异示例**（设 `decimals = 2`）：

| 待舍入值 | 单项计算结果（decimals=2） | 最终汇总结果（decimals=0） | 差异 |
|---------|--------------------------|--------------------------|------|
| 12.34 | round(2.468, 2, HALF_EVEN) * 5 = 2.47 * 5 = 12.35 | round(2.468) * 5 = 2 * 5 = 10.00 | 差 2.35 |
| 10.10 | round(2.02, 2, HALF_EVEN) * 5 = 2.02 * 5 = 10.10 | round(2.02) * 5 = 2 * 5 = 10.00 | 差 0.10 |

最终汇总的 HALF_FIVE 实现实际上是"舍入到最近的 5 的整数倍"，而非"舍入到指定小数位后再对齐 5 的倍数"。

#### 7.6.3 差异二：舍入模式

**单项计算**：使用 `Rounding_mode::HALF_EVEN` 作为内部舍入模式
**最终汇总**：使用 PHP `round()` 的默认模式 `PHP_ROUND_HALF_UP`

当除以 5 后的值恰好处于 0.5 边界时，两处行为不同：

| 待舍入值 | 除以 5 后 | 单项（HALF_EVEN） | 最终（HALF_UP） |
|---------|----------|------------------|----------------|
| 12.50 | 2.5 | 2（偶数） → 10.00 | 3（四舍五入） → 15.00 |
| 7.50 | 1.5 | 2（偶数） → 10.00 | 2（四舍五入） → 10.00 |
| 17.50 | 3.5 | 4（偶数） → 20.00 | 4（四舍五入） → 20.00 |

当除以 5 后的整数部分为奇数且小数部分恰好为 0.5 时，两处结果会不同。

#### 7.6.4 差异三：输入类型与精度传播

- 单项计算中，`$amount` 是 `float` 类型（`get_tax_for_amount` 中 bcmul 的结果为字符串，经隐式转换传入）
- 最终汇总中，`$tax_amount` 是通过 `bcadd(..., 4)` 累加的字符串，转换为 float 后参与计算

在极端精度场景下，浮点精度可能引入额外差异，但这不是主要差异来源。

### 7.7 舍入不一致性的影响范围

| 舍入模式 | 基础税制 | 目的地税制 |
|---------|---------|-----------|
| HALF_UP | 一致（都用 HALF_UP） | 一致 |
| HALF_DOWN | 不涉及 | 一致（都用 PHP 原生 round） |
| HALF_EVEN | 不涉及 | 一致（都用 PHP 原生 round） |
| **HALF_ODD** | 不涉及 | **不一致**（汇总处用了 HALF_UP） |
| ROUND_UP | 不涉及 | 基本一致（实现思路相同，写法略有差异） |
| ROUND_DOWN | 不涉及 | 基本一致 |
| **HALF_FIVE** | 不涉及 | **不一致**（小数位、舍入模式均不同） |

> **注意**：`apply_invoice_taxing()` 方法调用的是 `get_tax_for_amount()`，因此与单项计算使用同一套实现。但该方法目前仅用于数据迁移，不影响正常销售流程。

---

## 8. 税种分组与显示顺序

税额按 `tax_rate% + tax_group` 聚合，分组键生成见 [Tax_lib::update_taxes()](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Libraries/Tax_lib.php#L196)：

```php
$tax_group_index = $this->clean('X' . (float)$tax_rate . '% ' . $tax_group);
```

显示/打印顺序由 `print_sequence` 字段决定，最终舍入前会按此字段排序，见 [Tax_lib::round_taxes()](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Libraries/Tax_lib.php#L254-L260)。

目的地税制下 `print_sequence` 的值为：
```
tax_jurisdictions.tax_group_sequence + tax_categories.tax_group_sequence
```
见 [Tax::get_taxes()](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Models/Tax.php#L120)。

---

## 9. 配置项总览

| 配置项 | 文件位置 | 说明 |
|--------|---------|------|
| `tax_included` | [tax_config.php](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Views/configs/tax_config.php#L31-L40) | 全局价内/价外税标记（基础税制） |
| `default_tax_1_name` | [tax_config.php](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Views/configs/tax_config.php#L43-L61) | 默认税种 1 名称（基础税制） |
| `default_tax_1_rate` | [tax_config.php](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Views/configs/tax_config.php#L43-L61) | 默认税率 1（基础税制） |
| `default_tax_2_name` | [tax_config.php](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Views/configs/tax_config.php#L63-L82) | 默认税种 2 名称（基础税制） |
| `default_tax_2_rate` | [tax_config.php](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Views/configs/tax_config.php#L63-L82) | 默认税率 2（基础税制） |
| `use_destination_based_tax` | [tax_config.php](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Views/configs/tax_config.php#L84-L94) | 启用目的地税制 |
| `default_tax_code` | [tax_config.php](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Views/configs/tax_config.php#L96-L106) | 默认税码（目的地税制） |
| `default_tax_category` | [tax_config.php](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Views/configs/tax_config.php#L108-L118) | 默认税类（目的地税制） |
| `default_tax_jurisdiction` | [tax_config.php](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Views/configs/tax_config.php#L120-L130) | 默认税务辖区 |
| `tax_decimals` | [locale_helper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Helpers/locale_helper.php#L327-L331) | 税金额小数位 |
| `currency_decimals` | [locale_helper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/17-opensourcepos/app/Helpers/locale_helper.php#L309-L313) | 货币金额小数位 |

---

## 10. 计税流程图

```
开始
  │
  ▼
客户判定：散客 或 taxable=1 ?
  ├─ 否 → 全部税额 = 0，结束
  └─ 是
       │
       ▼
use_destination_based_tax ?
  ├─────────────┐
  否             是
  │              │
  ▼              ▼
取 items_taxes    确定税码
按商品逐项计税     │
  │              ▼
  │           确定税类
  │              │
  │              ▼
  │           查询 tax_rates
  │              │
  │              ▼
  │           按级次逐项计税
  │              │
  └──────┬───────┘
         │
         ▼
   按税种分组累加（4 位小数精度）
         │
         ▼
   统一舍入（tax_included ? tax_decimals : currency_decimals）
         │
         ▼
   返回 taxes[] + item_taxes[]
```
