# 销售结账流程代码分析

## 一、整体架构

销售结账流程涉及三层架构：

| 层级 | 文件 | 职责 |
|------|------|------|
| 控制器层 | [Sales.php](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Controllers/Sales.php) | 处理 HTTP 请求，协调业务逻辑 |
| 业务逻辑层 | [Sale_lib.php](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Sale_lib.php) | 购物车、折扣、支付、总额计算 |
| 数据访问层 | [Sale.php](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Models/Sale.php) | 数据库读写操作 |
| 税费计算 | [Tax_lib.php](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Tax_lib.php) | 税费计算逻辑 |

---

## 二、完整流程路径

### 2.1 流程总览

```
用户进入收银台 (getIndex)
        ↓
加载购物车 & 计算显示 (_reload)
        ↓
添加商品 (postAdd) → sale_lib->add_item()
        ↓
编辑商品/删除商品 → sale_lib->edit_item() / delete_item()
        ↓
选择客户 (postSelectCustomer) → 应用客户折扣
        ↓
添加支付 (postAddPayment) → sale_lib->add_payment()
        ↓
完成结账 (postComplete)
        ├─ 计算税费 tax_lib->get_taxes()
        ├─ 计算总额 sale_lib->get_totals()
        ├─ 写入数据库 sale->save_value()
        │   ├─ 事务开始
        │   ├─ 插入 sales 表
        │   ├─ 插入 sales_payments 表
        │   ├─ 插入 sales_items 表
        │   ├─ 更新库存
        │   ├─ 插入 sales_taxes / sales_items_taxes 表
        │   └─ 事务提交
        └─ 返回收据视图
```

### 2.2 入口：收银台页面

**控制器方法**：[Sales::getIndex()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Controllers/Sales.php#L69-L73)

```php
public function getIndex(): ResponseInterface|string
{
    $this->session->set('allow_temp_items', 1);
    return $this->_reload();
}
```

调用 `_reload()` 方法渲染收银台页面，该方法是所有操作后刷新页面的统一入口。

### 2.3 页面刷新：_reload 方法

**控制器方法**：[Sales::_reload()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Controllers/Sales.php#L1188-L1280)

主要职责：
1. 重置现金四舍五入标志
2. 获取购物车数据
3. 加载客户信息
4. 计算税费（`tax_lib->get_taxes()`）
5. 计算总额（`sale_lib->get_totals()`）
6. 获取支付列表
7. 传递所有数据到视图

---

## 三、购物车管理

### 3.1 购物车数据结构

购物车存储在 Session 中，键名为 `sales_cart`，是一个以行号 (line) 为键的数组。

每个商品项包含：

```php
[
    'item_id'               => int,      // 商品ID
    'item_location'         => int,      // 仓库位置ID
    'line'                  => int,      // 行号
    'name'                  => string,   // 商品名称
    'item_number'           => string,   // 商品编号
    'description'           => string,   // 描述
    'serialnumber'          => string,   // 序列号
    'quantity'              => string,   // 数量
    'discount'              => string,   // 折扣值
    'discount_type'         => int,      // 折扣类型: 0=百分比, 1=固定金额
    'price'                 => string,   // 单价
    'cost_price'            => string,   // 成本价
    'total'                 => string,   // 小计 (未折)
    'discounted_total'      => string,   // 折后小计
    'print_option'          => int,      // 打印选项
    'stock_type'            => int,      // 库存类型: 0=有库存, 1=无库存
    'item_type'             => int,      // 商品类型
    'tax_category_id'       => int,      // 税类别ID
    // ...
]
```

### 3.2 添加商品

**控制器方法**：[Sales::postAdd()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Controllers/Sales.php#L503-L575)

**业务逻辑**：[Sale_lib::add_item()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Sale_lib.php#L1034-L1186)

流程：
1. 获取默认折扣（配置或客户折扣）
2. 判断是普通商品、套装商品还是退货
3. 调用 `sale_lib->add_item()` 添加到购物车
4. 如果商品已存在且无需序列号，则累加数量
5. 重新计算行号、小计等

### 3.3 编辑商品

**控制器方法**：[Sales::postEditItem()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Controllers/Sales.php#L584-L647)

**业务逻辑**：[Sale_lib::edit_item()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Sale_lib.php#L1259-L1284)

验证规则：
- 价格必须为非负小数
- 数量必须为小数（退货允许负数）
- 百分比折扣不能超过 100%
- 固定折扣不能超过商品总价

---

## 四、折扣计算逻辑

### 4.1 折扣类型

定义在 [Constants.php](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Config/Constants.php#L146-L147)：

```php
const PERCENT = 0;  // 百分比折扣
const FIXED   = 1;  // 固定金额折扣
```

### 4.2 折扣来源

1. **系统默认折扣**：`$this->config['default_sales_discount']`
2. **客户折扣**：客户表中的 `discount` 和 `discount_type` 字段
3. **商品套装折扣**：`item_kit` 表中的 `kit_discount` 和 `kit_discount_type`
4. **手动编辑折扣**：在收银台逐行修改

折扣比较规则（详见 [4.6 节](#46-折扣优先级比较代码事实对齐版--无冲突结论)）：
- **客户折扣 > 系统默认折扣**：无条件覆盖
- **套装折扣 vs 基准折扣**：类型相同时取较大值，类型不同时套装折扣覆盖
- 操作顺序影响最终结果：先选客户后加商品 vs 先加商品后选客户，折扣归属不同

### 4.3 折扣计算方法

**核心方法**：[Sale_lib::get_item_discount()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Sale_lib.php#L1623-L1633)

```php
public function get_item_discount(string $quantity, string $price, string $discount, int $discount_type): string
{
    $total = bcmul($quantity, $price);
    if ($discount_type == PERCENT) {
        $discount = bcmul($total, bcdiv($discount, '100'));
    } else {
        $discount = bcmul($quantity, $discount);
    }
    return (string)round((float)$discount, totals_decimals(), PHP_ROUND_HALF_UP);
}
```

计算逻辑：
- **百分比折扣**：`数量 × 单价 × 折扣%`
- **固定折扣**：`数量 × 每单位折扣金额`

### 4.4 折扣应用时机

1. **添加商品时**：根据当前配置/客户折扣应用初始折扣
2. **选择客户时**：[Sales::postSelectCustomer()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Controllers/Sales.php#L231-L246) 调用 `sale_lib->apply_customer_discount()` 将客户折扣应用到折扣为 0 的商品
3. **手动编辑时**：直接修改单行折扣

### 4.5 总折扣计算

**方法**：[Sale_lib::get_discount()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Sale_lib.php#L1501-L1512)

遍历购物车中所有折扣大于 0 的商品，累加折扣金额。

### 4.6 折扣优先级比较（代码事实对齐版 · 无冲突结论）

折扣优先级不是简单的「套装 > 客户 > 系统默认」线性关系，而是分两个阶段、受操作顺序和折扣类型影响的复杂逻辑。所有结论均严格对齐代码事实。

---

#### 先总括：折扣来源与处理流程

**折扣的三个来源**：

| 来源 | 配置位置 | 说明 |
|------|----------|------|
| 系统默认折扣 | `config['default_sales_discount']` | 无客户时的基准折扣 |
| 客户折扣 | `customer.discount` | 特定客户享受的折扣，有折扣类型 |
| 套装折扣 | `item_kit.kit_discount` | 购买套装时享受的折扣，有折扣类型 |

**两个核心处理阶段**：

```
阶段一：添加商品时（postAdd）
    ├─ 步骤1：确定基准折扣
    │   ├─ 有客户且客户有折扣 → 基准折扣 = 客户折扣
    │   └─ 无客户或客户无折扣 → 基准折扣 = 系统默认折扣
    └─ 步骤2：如果是套装商品 → 基准折扣 与 套装折扣 比较
        ├─ 类型相同 → 取较大值
        └─ 类型不同 → 用套装折扣（无条件覆盖）

阶段二：选择客户时（apply_customer_discount）
    └─ 只对 discount == 0 的商品应用客户折扣
       （已有折扣的商品不被覆盖）
```

**操作顺序决定最终结果**：
- **先选客户，后加商品** → 客户折扣在阶段一参与比较，可能与套装折扣竞争
- **先加商品，后选客户** → 客户折扣在阶段二回溯应用，只能覆盖无折扣商品

---

#### 阶段一：添加商品时的比较逻辑（postAdd）

**位置**：[Sales::postAdd()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Controllers/Sales.php#L507-L547)

**步骤1：确定基准折扣**

```php
// 初始值：系统默认折扣
$discount = $this->config['default_sales_discount'];
$discount_type = $this->config['default_sales_discount_type'];

// 如已选客户且客户有折扣 → 替换为客户折扣
$customer_id = $this->sale_lib->get_customer();
if ($customer_id != NEW_ENTRY) {
    $customer_discount = $this->customer->get_info($customer_id)->discount;
    $customer_discount_type = $this->customer->get_info($customer_id)->discount_type;
    if ($customer_discount != '') {
        $discount = $customer_discount;        // 覆盖
        $discount_type = $customer_discount_type;
    }
}
```

**代码事实**：客户折扣**无条件覆盖**系统默认折扣（只要客户有折扣）。这是真正的优先级关系：**客户折扣 > 系统默认折扣**。

---

**步骤2：套装商品时与套装折扣比较**

```php
if ($this->item_kit->is_valid_item_kit(...)) {
    $item_kit_info = $this->item_kit->get_info($item_kit_id);

    if ($discount_type == $item_kit_info->kit_discount_type) {
        // 折扣类型相同 → 取数值大的（折扣越大越优惠）
        if ($item_kit_info->kit_discount > $discount) {
            $discount = $item_kit_info->kit_discount;
        }
    } else {
        // 折扣类型不同 → 无条件使用套装折扣
        $discount = $item_kit_info->kit_discount;
        $discount_type = $item_kit_info->kit_discount_type;
    }
}
```

**⚠️ 精确比较规则（无冲突版）**：

| 基准折扣来源 | 基准折扣类型 | 套装折扣类型 | 比较规则 | 结果 |
|-------------|-------------|-------------|----------|------|
| 系统默认 / 客户 | 百分比 (0) | 百分比 (0) | 取较大值 | `max(基准, 套装)` |
| 系统默认 / 客户 | 固定金额 (1) | 固定金额 (1) | 取较大值 | `max(基准, 套装)` |
| 系统默认 / 客户 | 百分比 (0) | 固定金额 (1) | 类型不同 → 套装覆盖 | 套装折扣 |
| 系统默认 / 客户 | 固定金额 (1) | 百分比 (0) | 类型不同 → 套装覆盖 | 套装折扣 |

**之前的错误说法修正**：
| 之前的错误说法 | 正确的代码事实 |
|---------------|---------------|
| 「套装折扣优先级最高」 | ❌ 错。类型相同时取较大值，客户折扣可能更大 |
| 「无客户折扣时系统默认 < 套装取套装」 | ❌ 冗余表述。这就是「类型相同取较大值」的特例 |
| 「线性优先级：套装 > 客户 > 系统默认」 | ❌ 错。客户 > 系统默认是无条件的；套装与基准的比较取决于类型 |

**正确的优先级关系**：
1. **客户折扣 > 系统默认折扣**（无条件覆盖）
2. **套装折扣 vs 基准折扣**：类型相同时取较大值，类型不同时套装覆盖

---

#### 阶段二：折扣在套装内的消耗（PRICE_MODE_KIT）

**位置**：[Sale_lib::add_item()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Sale_lib.php#L1055-L1078)

当 `$price_mode == PRICE_MODE_KIT` 时，折扣会在套装各组件间分摊消耗。这里的关键是**传值/传引用导致的独立折扣池**：

**函数签名对比**：

| 调用位置 | 函数签名 | `$discount` 传递方式 | 折扣消耗是否影响外层 |
|----------|----------|-------------------|----------------------|
| `postAdd` 添加套装头 | `add_item(string &$item_id, ..., string &$discount, ...)` | **引用传参** | ✅ 影响（修改 `postAdd` 中的 `$discount`） |
| `postAdd` 调用 `add_item_kit` | `add_item_kit(..., float $discount, ...)` | **传值** | ❌ 不影响（`add_item_kit` 获得副本） |
| `add_item_kit` 内循环添加组件 | `add_item(..., string &$discount, ...)` | **引用传参** | ✅ 仅内部影响（修改 `add_item_kit` 中的副本） |

**完整调用链与折扣消耗**：

```
postAdd 中的 $discount = 10.00（已比较后的最终折扣）
    │
    ├─ 调用 add_item(套装头, &$discount, ...) → 引用传参
    │   └─ PRICE_MODE_KIT 下消耗折扣
    │       └─ 如果是 FIXED 类型且折扣 > 单价
    │           ├─ 实际折扣 = 单价
    │           └─ $discount -= 实际折扣 → 修改 postAdd 中的 $discount（比如变成 7.00）
    │
    └─ 调用 add_item_kit(..., $discount = 7.00, ...) → 传值（获得当前值的副本）
        └─ 循环调用 add_item(每个组件, &$discount_copy, ...)
            └─ 折扣消耗修改的是副本，不影响 postAdd 中的 $discount
```

**代码事实**：套装头和套装组件使用**独立的折扣池**。套装头消耗后剩余的折扣传给组件，组件消耗的是副本，不会反向影响外层。

**FIXED 类型折扣的消耗逻辑**：

```php
if ($discount_type == FIXED) {
    if ($applied_discount > $price) {
        $applied_discount = $price;      // 单件折扣上限 = 单价
        $discount -= $applied_discount;  // 余额留给后续商品（引用传参）
    } else {
        $discount = 0;                   // 折扣已全部用完
    }
}
```

---

#### 阶段三：选择客户时的回溯应用（apply_customer_discount）

**位置**：[Sale_lib::apply_customer_discount()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Sale_lib.php#L1478-L1496)

```php
public function apply_customer_discount(string $discount, int $discount_type): void
{
    $items = $this->get_cart();
    foreach ($items as &$item) {
        // 只对 discount == 0 的商品应用客户折扣
        if ($item['discount'] == 0.0) {
            $item['discount'] = $discount;
            $item['discount_type'] = $discount_type;
            // 重新计算 total 和 discounted_total
        }
    }
    $this->set_cart($items);
}
```

**代码事实**：`apply_customer_discount` 是**保护性应用**，只覆盖无折扣的商品。这意味着：
- 已有折扣的商品（套装折扣、手动折扣等）**不会被覆盖**
- 这是「先加商品后选客户」时客户折扣无法影响套装商品的原因

---

#### 操作顺序对最终折扣的影响（完整对比）

**场景 A：先选客户，后加套装商品**

```
1. 选择客户（折扣 15%，PERCENT）
2. 添加套装商品（套装折扣 10%，PERCENT）
3. postAdd 中比较：类型相同 → 取较大值 → max(15%, 10%) = 15%
4. 结果：客户折扣生效，套装商品享受 15% 折扣
```

**场景 B：先加套装商品，后选客户**

```
1. 添加套装商品（套装折扣 10%，PERCENT）
2. postAdd 中比较：基准是系统默认（假设 0%）→ 取较大值 → 10%
3. 选择客户（折扣 15%，PERCENT）
4. apply_customer_discount：套装商品 discount != 0 → 不覆盖
5. 结果：套装商品仍享受 10% 折扣，客户折扣只应用于无折扣商品
```

**场景 C：类型不同时，套装折扣无条件覆盖**

```
1. 选择客户（折扣 20%，PERCENT）
2. 添加套装商品（套装折扣 5 元，FIXED）
3. postAdd 中比较：类型不同 → 套装覆盖 → 5 元 FIXED
4. 结果：即使客户折扣力度更大，套装折扣仍生效
```

**所有场景对比表**：

| 场景 | 操作顺序 | 基准折扣 | 套装折扣 | 类型 | 结果 | 谁赢 |
|------|----------|----------|----------|------|------|------|
| 1 | 先选客户后加套装 | 15%（客户） | 10%（套装） | 相同（PERCENT） | 15% | 客户 |
| 2 | 先选客户后加套装 | 10%（客户） | 15%（套装） | 相同（PERCENT） | 15% | 套装 |
| 3 | 先选客户后加套装 | 15%（客户） | 5元（套装） | 不同 | 5元（FIXED） | 套装（类型不同无条件覆盖） |
| 4 | 先加套装后选客户 | 0%（系统） | 10%（套装） | 相同（PERCENT） | 10% | 套装（客户折扣不回溯覆盖） |
| 5 | 先选客户后加普通商品 | 15%（客户） | - | - | 15% | 客户 |
| 6 | 先加普通商品后选客户 | 0%（系统） | - | - | 15%（应用于 discount==0 的商品） | 客户 |

---

#### 最终结论（无冲突版）

1. **客户折扣 > 系统默认折扣**：无条件覆盖（只要客户有折扣）
2. **套装折扣 vs 基准折扣**：
   - 类型相同 → 取较大值（谁更优惠用谁）
   - 类型不同 → 套装折扣无条件覆盖
3. **操作顺序影响结果**：
   - 先选客户后加商品 → 客户折扣在比较阶段参与竞争
   - 先加商品后选客户 → 客户折扣只能覆盖无折扣商品
4. **套装折扣消耗**：套装头和组件使用独立折扣池（传值 vs 传引用）
5. **`apply_customer_discount` 保护性应用**：不覆盖已有折扣

**没有所谓的「套装折扣优先级最高」，一切取决于折扣类型和操作顺序。**

## 五、税费计算逻辑

### 5.1 税费类型

定义在 [Tax_lib.php](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Tax_lib.php#L23-L24)：

```php
public const TAX_TYPE_EXCLUDED = '1';  // 价外税 (销售税)
public const TAX_TYPE_INCLUDED = '0';  // 价内税 (增值税/VAT)
```

### 5.2 税费计算入口

**核心方法**：[Tax_lib::get_taxes()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Tax_lib.php#L86-L171)

返回值是一个包含两个元素的数组：
- `$tax_details[0]`：税费汇总（按税组分类，用于 `sales_taxes` 表）
- `$tax_details[1]`：商品行级税费明细（用于 `sales_items_taxes` 表）

### 5.3 计税模式

#### 模式一：基础系统税 (use_destination_based_tax = false)

直接从 `item_taxes` 表获取商品关联的税率，逐行计算。

#### 模式二：目的地税 (use_destination_based_tax = true)

根据客户所在城市/州查找适用的税码，再结合商品税类别计算税率。

**方法**：[Tax_lib::apply_destination_tax()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Tax_lib.php#L300-L364)

### 5.4 价内税计算

**方法**：[Tax_lib::get_included_tax()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Tax_lib.php#L183-L189)

```php
public function get_included_tax($quantity, $price, $discount_percentage, $discount_type, $tax_percentage, ...): string
{
    $item_total = $this->get_item_total($quantity, $price, $discount_percentage, $discount_type, true);
    $tax_fraction = bcdiv(bcadd('100', $tax_percentage), '100');
    $price_tax_excl = bcdiv($item_total, $tax_fraction);
    return bcsub($item_total, $price_tax_excl);
}
```

公式推导：
- 含税价 = 不含税价 × (1 + 税率%)
- 不含税价 = 含税价 / (1 + 税率%)
- 税额 = 含税价 - 不含税价

### 5.5 价外税计算

**方法**：[Tax_lib::get_tax_for_amount()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Tax_lib.php#L76-L81)

```php
public function get_tax_for_amount(string $tax_basis, string $tax_percentage, int $rounding_mode, int $decimals): string
{
    $tax_amount = bcmul($tax_basis, bcdiv($tax_percentage, '100'));
    return rounding_mode::round_number($rounding_mode, $tax_amount, $decimals);
}
```

公式：`税额 = 计税基础 × 税率%`

### 5.6 级计税 (Cascade Tax)

在目的地税模式下，支持级计税：
- 后一段税的计税基础 = 前一段税的计税基础 + 前一段税的税额
- 通过 `cascade_sequence` 字段控制级次顺序

### 5.7 税费舍入规则

**方法**：[Tax_lib::round_taxes()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Tax_lib.php#L252-L294)

支持多种舍入模式：
- `HALF_UP`：四舍五入
- `HALF_DOWN`：五舍六入
- `HALF_EVEN`：银行家舍入
- `ROUND_UP`：向上取整
- `ROUND_DOWN`：向下取整
- `HALF_FIVE`：以 5 为基数舍入

### 5.8 客户免税

如果客户设置为非应税 (`taxable = false`)，则整单不计税。

---

## 六、多支付方式处理

### 6.1 支持的支付方式

定义在 [locale_helper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Helpers/locale_helper.php#L239-L279) 的 `get_payment_options()` 函数：

| 支付方式 | 说明 |
|----------|------|
| Cash (现金) | 现金支付 |
| Debit (借记卡) | 借记卡 |
| Credit (信用卡) | 信用卡 |
| Due (赊账) | 记到客户应收账款 |
| Check (支票) | 支票支付 |
| UPI | 印度统一支付接口 (条件启用) |
| Bank Transfer (银行转账) | 银行转账 |
| Wallet (电子钱包) | 电子钱包 |
| Gift Card (礼品卡) | 礼品卡支付 |
| Rewards (积分) | 积分抵扣 |

支付方式的显示顺序由配置 `payment_options_order` 决定。

### 6.2 支付数据结构

支付记录存储在 Session 的 `sales_payments` 中，以支付方式 ID 为键：

```php
[
    'payment_type'    => string,  // 支付类型名称
    'payment_amount'  => string,  // 支付金额
    'cash_refund'     => float,   // 现金找零
    'cash_adjustment' => int,     // 是否现金调整: 1=是, 0=否
]
```

### 6.3 添加支付

**控制器方法**：[Sales::postAddPayment()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Controllers/Sales.php#L393-L479)

**业务逻辑**：[Sale_lib::add_payment()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Sale_lib.php#L585-L612)

#### 特殊支付类型处理：

**礼品卡支付**：
- 输入的金额作为礼品卡号
- 验证礼品卡余额和归属客户
- 支付金额 = min(应付金额, 礼品卡余额)
- 礼品卡号码附加在支付类型后：`Giftcard:12345`

**积分支付**：
- 验证客户积分余额
- 支付金额 = min(应付金额, 积分点数)
- 积分余额会相应扣减

**现金支付**：
- 如果启用现金四舍五入，会自动添加一笔现金调整 (cash_adjustment)
- 现金调整用于平衡四舍五入造成的差额

### 6.4 删除支付

**方法**：[Sale_lib::delete_payment()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Sale_lib.php#L636-L655)

如果删除现金支付且启用了现金四舍五入，会同时删除现金调整支付。

### 6.5 支付总额计算

**方法**：[Sale_lib::get_payments_total()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Sale_lib.php#L669-L688)

- 排除现金调整金额
- 同时判断是否仅含现金类支付，以决定是否启用现金模式 (cash_mode)

### 6.6 多支付状态切换与付款清空触发时机（深度分析 · 代码事实版）

付款记录存储在 Session 的 `sales_payments` 键中，通过 `empty_payments()` 方法一次性清空。以下是所有触发路径的代码级分析。

#### 触发时机总览（完整代码路径）

| 场景 | 触发方法 | 实际调用 | 清空范围 | 设计原因 |
|------|----------|----------|----------|----------|
| 切换销售模式/仓库 | [Sales::postChangeMode()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Controllers/Sales.php#L293) | `sale_lib->empty_payments()` | **全部付款** | 模式/位置切换可能改变税率、折扣、支付方式可用性 |
| 编辑购物车商品 | [Sales::postEditItem()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Controllers/Sales.php#L638) | `sale_lib->empty_payments()` | **全部付款** | 商品价格/数量/折扣变化会改变应付总额，已录入的支付可能不匹配 |
| 删除购物车商品 | [Sales::getDeleteItem()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Controllers/Sales.php#L661) | `sale_lib->empty_payments()` | **全部付款** | 应付总额减少，可能出现多付 |
| 移除客户 | [Sales::getRemoveCustomer()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Controllers/Sales.php#L676) | `sale_lib->delete_payment('Rewards')` | **仅积分支付** | 积分与客户绑定，移除客户需清除积分支付 |
| 完成结账清场 | [Sale_lib::clear_all()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Sale_lib.php#L1426) | `empty_payments()` 内含 | **全部付款** | 整单结清，清空所有会话状态 |
| 加载历史销售 | [Sales::_load_sale_data()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Controllers/Sales.php#L1084) | `clear_all()` 内含 | **全部付款** | 重新开始新一笔交易 |
| 单据输出完成后 | [Sales::postComplete()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Controllers/Sales.php#L827/L861/L888/L919/L971) | `clear_all()` 内含 | **全部付款** | 收据/发票/邮件发送完成 |
| 每次页面刷新 | [Sales::_reload()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Controllers/Sales.php#L1196) | `reset_cash_rounding()` | **现金模式标志** | 每次刷新重置 cash_mode，再根据付款类型重新判断 |

**明确不会清空付款的操作**：
- `postAdd`（添加商品）
- `postSelectCustomer`（选择/切换客户）
- `postSetComment`（设置备注）
- `postSetInvoiceNumber`（设置发票号）
- `getDeletePayment`（删除单笔支付，只删指定那笔）

#### 各场景代码级详细说明

**场景1：切换销售模式或仓库位置 — 全额清空**

```php
public function postChangeMode(): ResponseInterface|string
{
    $mode = $this->request->getPost('mode', FILTER_SANITIZE_FULL_SPECIAL_CHARS);
    $this->sale_lib->set_mode($mode);

    // ... 设置 sale_type、餐桌 ...

    $stock_location = $this->request->getPost('stock_location', FILTER_SANITIZE_NUMBER_INT);
    if ($this->stock_location->is_allowed_location($stock_location, 'sales')) {
        $this->sale_lib->set_sale_location($stock_location);
    }

    $this->sale_lib->empty_payments();   // ← 无条件清空所有付款
    return $this->_reload();
}
```

清空原因：
- 切换到退货模式后，数量变负，税费计算方向改变
- 切换到报价单/工单后，积分、礼品卡等支付方式不可用
- 切换仓库后，商品单价和库存可能不同

**场景2：编辑购物车中的商品 — 全额清空**

```php
public function postEditItem(): ResponseInterface|string
{
    // ... 验证 quantity / price / discount ...
    $this->sale_lib->edit_item($line, $description, $serialnumber, $quantity, $discount, $discount_type, $price, $discounted_total);

    $this->sale_lib->empty_payments();   // ← 编辑后无条件清空所有付款
    return $this->_reload($data);
}
```

清空原因：任何参数变化都会改变应付总额，已录入的支付可能不再匹配。

**场景3：删除购物车商品 — 全额清空**

```php
public function getDeleteItem(int $item_id): ResponseInterface|string
{
    $this->sale_lib->delete_item($item_id);
    $this->sale_lib->empty_payments();   // ← 删除后无条件清空所有付款
    return $this->_reload();
}
```

清空原因：删除商品直接减少应付总额，可能出现多付。

**场景4：移除客户 — 仅清积分支付，其余付款保留**

这是最容易混淆的场景。看代码：

```php
public function getRemoveCustomer(): ResponseInterface|string
{
    $this->sale_lib->clear_giftcard_remainder();
    $this->sale_lib->clear_rewards_remainder();
    $this->sale_lib->delete_payment(lang('Sales.rewards'));   // ← 只删积分支付！
    $this->sale_lib->clear_invoice_number();
    $this->sale_lib->clear_quote_number();
    $this->sale_lib->remove_customer();                       // ← 只清 session 中的 customer_id

    return $this->_reload();
}
```

**关键点（与代码事实对齐）**：
- 调用 `delete_payment('Rewards')` — 只删除积分这一种支付方式的记录
- 不调用 `empty_payments()` — 现金、刷卡、礼品卡等其他支付**全部保留**
- `remove_customer()` 仅删除 Session 的 `sales_customer` 键，不碰付款数组
- 礼品卡只清「待支付余额」(`giftcard_remainder`)，已录入的礼品卡支付**不会被删除**

**设计原因**：积分是客户专属的，换客户后原客户的积分不能用；但现金、刷卡等支付方式与客户无关，可以保留。

**场景5：选择/切换客户 — 付款完全保留**

```php
public function postSelectCustomer(): ResponseInterface|string
{
    $customer_id = (int)$this->request->getPost('customer', FILTER_SANITIZE_NUMBER_INT);
    if ($this->customer->exists($customer_id)) {
        $this->sale_lib->set_customer($customer_id);
        $discount = $this->customer->get_info($customer_id)->discount;
        $discount_type = $this->customer->get_info($customer_id)->discount_type;

        if ($discount != '') {
            $this->sale_lib->apply_customer_discount($discount, $discount_type);
        }
    }
    // 注意：全程没有任何清空付款的操作！
    return $this->_reload();
}
```

**为什么不清空？**
- `apply_customer_discount()` 只覆盖 `discount == 0` 的商品，不会减少已有折扣
- 应付可能减少，但实际业务中选客户是常规操作，强制重输付款体验差
- **风险兜底**：如果应付减少导致多付，在 `postComplete` 时会通过 `cash_refund`（找零）处理

**场景6：每次页面刷新 — 重置现金模式标志**

这是一个隐式的状态切换：

```php
private function _reload(array $data = []): ResponseInterface|string
{
    $cash_rounding = $this->sale_lib->reset_cash_rounding();
    // ...
}
```

在 [`reset_cash_rounding()`](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Sale_lib.php#L1446-L1459) 中：
```php
public function reset_cash_rounding(): int
{
    // ... 检查是否启用现金四舍五入 ...
    $this->session->set('cash_rounding', $cash_rounding);
    $this->session->set('cash_mode', CASH_MODE_FALSE);   // ← 每次刷新强制置为 false

    return $cash_rounding;
}
```

然后在 [`get_payments_total()`](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Sale_lib.php#L669-L688) 中重新判断：
```php
public function get_payments_total(): string
{
    $cash_mode_eligible = CASH_MODE_TRUE;
    foreach ($this->get_payments() as $payments) {
        if (lang('Sales.cash') != $payments['payment_type']
            && lang('Sales.cash_adjustment') != $payments['payment_type']) {
            $cash_mode_eligible = CASH_MODE_FALSE;   // 有非现金支付 → 不进入现金模式
        }
    }
    if ($cash_mode_eligible && $this->session->get('cash_rounding')) {
        $this->session->set('cash_mode', CASH_MODE_TRUE);   // 重新置为 true
    }
    // ...
}
```

**状态切换逻辑**：每次刷新页面时，`cash_mode` 先被重置为 `false`，再根据当前付款数组中是否只有现金类支付来重新判断是否进入现金模式。

#### 付款部分修改的场景

**删除单笔支付**：

```php
public function getDeletePayment(string $payment_id): ResponseInterface|string
{
    $this->sale_lib->delete_payment(base64url_decode($payment_id));
    return $this->_reload();
}
```

在 [`delete_payment()`](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Sale_lib.php#L636-L655) 中：
```php
public function delete_payment(string $payment_id): void
{
    $payments = $this->get_payments();
    $decoded_payment_id = urldecode($payment_id);

    unset($payments[$decoded_payment_id]);   // 只删指定的那笔

    // 现金与现金调整绑定删除
    $cash_rounding = $this->reset_cash_rounding();
    if ($cash_rounding) {
        if ($decoded_payment_id == lang('Sales.cash')) {
            unset($payments[lang('Sales.cash_adjustment')]);
        }
        if ($decoded_payment_id == lang('Sales.cash_adjustment')) {
            unset($payments[lang('Sales.cash')]);
        }
    }
    $this->set_payments($payments);
}
```

**添加新支付时的现金模式切换**：

在 [`add_payment()`](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Sale_lib.php#L605-L609) 中：
```php
if ($this->session->get('cash_mode')) {
    // 已在现金模式，但新添加非现金支付 → 退出现金模式
    if ($this->session->get('cash_rounding')
        && $payment_id != lang('Sales.cash')
        && $payment_id != lang('Sales.cash_adjustment')) {
        $this->session->set('cash_mode', CASH_MODE_FALSE);
        // 注意：已有的现金调整记录不会被删除！
        // 但后续计算将从 cash_amount_due 切回 amount_due
    }
}
```

**关键发现**：退出现金模式时**不会删除**已录入的现金调整记录，这可能导致结账时出现异常找零。

---

#### 设计权衡分析

| 操作类型 | 清空策略 | 风险 | 收益 |
|----------|----------|------|------|
| 编辑/删除商品 | 全额清空 | 用户体验差（需重输付款） | 绝对保证数据一致性 |
| 添加商品 | 不清空 | 应付增加后可能付少了 | 流畅的用户体验 |
| 选择客户 | 不清空 | 应付减少后可能多付（通过找零兜底） | 流畅的用户体验 |
| 移除客户 | 仅清积分 | 客户折扣消失可能多付（通过找零兜底） | 保留与客户无关的付款 |
| 切换模式 | 全额清空 | 用户体验差 | 防止模式切换后的计算错误 |
| 页面刷新 | 仅重置 cash_mode | 现金模式可能短暂失效 | 确保现金模式判断准确 |

---

#### 清空范围总结

| 操作 | `empty_payments()` 调用 | 清空范围 |
|------|---------------------|----------|
| 编辑商品 | ✅ 是 | 全部付款 |
| 删除商品 | ✅ 是 | 全部付款 |
| 添加商品 | ❌ 否 | 无 |
| 选择客户 | ❌ 否 | 无 |
| 移除客户 | ❌ 否（调用 `delete_payment('Rewards')`） | 仅积分支付 |
| 切换模式 | ✅ 是 | 全部付款 |
| 页面刷新 | ❌ 否（调用 `reset_cash_rounding()`） | 仅 `cash_mode` 标志 |

---

## 七、总额计算

### 7.1 核心方法

**方法**：[Sale_lib::get_totals()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Sale_lib.php#L694-L783)

返回值包含：

| 字段 | 说明 |
|------|------|
| `prediscount_subtotal` | 折扣前小计 |
| `total_discount` | 总折扣金额 |
| `subtotal` | 折扣后小计 (不含价外税) |
| `tax_total` | 价外税总额 |
| `total` | 订单总额 (含所有税) |
| `payment_total` | 已支付总额 |
| `cash_total` | 现金模式下的总额 (四舍五入后) |
| `amount_due` | 应付金额 = 总额 - 已支付 |
| `cash_amount_due` | 现金模式下的应付金额 |
| `payments_cover_total` | 支付是否足以覆盖总额 |
| `item_count` | 有库存商品种类数 |
| `total_units` | 总数量 |
| `cash_adjustment_amount` | 现金调整金额 |

### 7.2 计算流程

```
1. 遍历购物车
   ├─ 累加折扣前总额 (prediscount_subtotal)
   ├─ 计算每项折扣，累加总折扣
   └─ 累加折后总额 (subtotal / total)

2. 处理税费
   ├─ 价外税 (EXCLUDED): 加到 total
   └─ 价内税 (INCLUDED): 从 subtotal 中扣除

3. 计算支付总额
   └─ payment_total = sum(非调整支付)

4. 现金四舍五入 (如启用)
   └─ cash_total = round(total)

5. 计算应付金额
   ├─ amount_due = total - payment_total
   └─ cash_amount_due = cash_total - payment_total

6. 判断支付是否足够
   └─ 考虑浮点精度阈值
```

### 7.3 现金四舍五入

当 `cash_decimals() < totals_decimals()` 或配置了特定舍入模式时启用。

**方法**：[Sale_lib::check_for_cash_rounding()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Sale_lib.php#L1718-L1724)

只有当所有支付都是现金类支付时，才会进入现金模式 (cash_mode = true)。

---

## 八、销售记录落库

### 8.1 入口方法

**控制器**：[Sales::postComplete()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Controllers/Sales.php#L691-L923)

**模型方法**：[Sale::save_value()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Models/Sale.php#L518-L688)

### 8.2 结账前校验

在 `postComplete()` 中进行的校验：

1. **负总额校验**：非退货模式下总额不能为负 (防欺诈/盗窃)
2. **发票号重复校验**：发票模式下检查发票号是否已存在
3. **报价单/工单号重复校验**

### 8.3 销售类型与状态

**销售类型** (sale_type)：

| 常量 | 值 | 说明 |
|------|----|------|
| `SALE_TYPE_POS` | 0 | 普通零售/收据 |
| `SALE_TYPE_INVOICE` | 1 | 发票 |
| `SALE_TYPE_WORK_ORDER` | 2 | 工单 |
| `SALE_TYPE_QUOTE` | 3 | 报价单 |
| `SALE_TYPE_RETURN` | 4 | 退货 |

**销售状态** (sale_status)：

| 常量 | 值 | 说明 |
|------|----|------|
| `COMPLETED` | 0 | 已完成 |
| `SUSPENDED` | 1 | 已挂起 (工单/报价单) |
| `CANCELED` | 2 | 已取消 |

### 8.4 数据库事务

整个保存过程在一个数据库事务中完成：

```php
$this->db->transStart();
// ... 所有数据库操作 ...
$this->db->transComplete();
return $this->db->transStatus() ? $sale_id : -1;
```

### 8.5 写入的表

#### 1. sales 表 - 销售主表

**字段**：

| 字段 | 说明 |
|------|------|
| `sale_id` | 销售ID (自增主键) |
| `sale_time` | 销售时间 |
| `customer_id` | 客户ID (可为空) |
| `employee_id` | 员工ID |
| `comment` | 备注 |
| `sale_status` | 销售状态 |
| `sale_type` | 销售类型 |
| `invoice_number` | 发票号 |
| `quote_number` | 报价单号 |
| `work_order_number` | 工单号 |
| `dinner_table_id` | 餐桌ID |

#### 2. sales_payments 表 - 支付记录

**方法**：遍历 `$payments` 数组逐条插入

**字段**：

| 字段 | 说明 |
|------|------|
| `payment_id` | 支付记录ID (自增) |
| `sale_id` | 销售ID |
| `payment_type` | 支付类型 |
| `payment_amount` | 支付金额 |
| `cash_refund` | 现金找零 |
| `cash_adjustment` | 是否现金调整 |
| `employee_id` | 收款员工ID |

**特殊处理**：
- 礼品卡支付：扣减礼品卡余额
- 积分支付：扣减客户积分

#### 3. sales_items 表 - 销售商品明细

**字段**：

| 字段 | 说明 |
|------|------|
| `sale_id` | 销售ID |
| `item_id` | 商品ID |
| `line` | 行号 |
| `description` | 描述 |
| `serialnumber` | 序列号 |
| `quantity_purchased` | 购买数量 |
| `discount` | 折扣值 |
| `discount_type` | 折扣类型 |
| `item_cost_price` | 成本价 |
| `item_unit_price` | 销售单价 |
| `item_location` | 仓库位置 |
| `print_option` | 打印选项 |

#### 4. 库存更新

只有当商品是有库存类型 (`stock_type == HAS_STOCK`) 且销售状态为已完成 (`COMPLETED`) 时才更新：

- 更新 `item_quantities` 表：减少库存数量
- 插入 `inventory` 表：记录库存变动流水
- 退货时 (`quantity < 0`)：恢复库存，恢复已删除商品

#### 5. sales_taxes 表 - 销售税费汇总

**方法**：[Sale::save_sales_tax()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Models/Sale.php#L693-L701)

按税组汇总的税费记录。

#### 6. sales_items_taxes 表 - 商品行级税费明细

**方法**：[Sale::save_sales_items_taxes()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Models/Sale.php#L710-L732)

每个商品每行的每个税种一条记录。

**字段**：

| 字段 | 说明 |
|------|------|
| `sale_id` | 销售ID |
| `item_id` | 商品ID |
| `line` | 行号 |
| `name` | 税种名称 |
| `percent` | 税率 |
| `tax_type` | 税类型 (价内/价外) |
| `rounding_code` | 舍入代码 |
| `cascade_sequence` | 级次顺序 |
| `item_tax_amount` | 税额 |
| `sales_tax_code_id` | 税码ID |
| `tax_category_id` | 税类别ID |
| `jurisdiction_id` | 税务管辖区ID |

### 8.6 客户积分

**方法**：`save_customer_rewards()` (在 save_value 中调用)

根据消费金额和积分使用情况，更新客户积分。

### 8.7 餐桌管理

如果启用了餐桌功能：
- 销售完成：释放餐桌 (`dinner_table->release()`)
- 销售挂起：占用餐桌 (`dinner_table->occupy()`)

### 8.8 保存后的操作

1. 清空购物车和支付：`$this->sale_lib->clear_all()`
2. 生成收据条形码
3. 返回对应视图：收据/发票/报价单/工单

### 8.9 现金找零与现金调整对落库的影响（深度分析）

现金支付涉及两个特殊字段：`cash_refund`（找零）和 `cash_adjustment`（现金调整）。它们影响支付记录落库、实际收款核算和客户积分计算。

#### 概念区分

| 字段 | 含义 | 产生时机 | 正负方向 |
|------|------|----------|----------|
| `cash_refund` | 现金找零，即实收现金与应付的差额 | `postComplete` 结账时 | 正值 = 退还给客户的金额 |
| `cash_adjustment` | 现金四舍五入调整，平衡精确应付与实际收取的现金面额差异 | `postAddPayment` 添加现金支付时 | 可正可负（正 = 多收，负 = 抹零） |

#### 现金调整的产生过程

**位置**：[Sales::postAddPayment()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Controllers/Sales.php#L462-L471)

```php
elseif ($payment_type === lang('Sales.cash')) {
    $amount_due = $this->sale_lib->get_total();       // 现金模式下的四舍五入后应付
    $sales_total = $this->sale_lib->get_total(false); // 精确计算的应付（未四舍五入）
    $amount_tendered = parse_decimals($this->request->getPost('amount_tendered'));

    $this->sale_lib->add_payment($payment_type, $amount_tendered);

    $cash_adjustment_amount = $amount_due - $sales_total;
    if ($cash_adjustment_amount <> 0) {
        $this->session->set('cash_mode', CASH_MODE_TRUE);
        $this->sale_lib->add_payment(
            lang('Sales.cash_adjustment'),
            $cash_adjustment_amount,
            CASH_ADJUSTMENT_TRUE    // 标记为调整记录
        );
    }
}
```

**计算逻辑**：
- `get_total()` 不带参数 → 内部用 `cash_mode` 判断是否调用 `check_for_cash_rounding()` 进行四舍五入
- `get_total(false)` → 强制不四舍五入，返回精确计算值
- 调整金额 = 四舍五入后应付 - 精确应付
- 调整金额不为 0 时，额外插入一条「现金调整」支付记录

**示例**：
- 精确应付 = 12.345 元
- 现金四舍五入后 = 12.35 元（cash_decimals = 2，HALF_UP）
- 调整金额 = 12.35 - 12.345 = 0.005 元 → 系统多收 0.5 分

#### 现金找零的产生过程

**位置**：[Sales::postComplete()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Controllers/Sales.php#L763-L787)

```php
if ($data['cash_mode']) {
    $data['amount_due'] = $totals['cash_amount_due'];    // 用四舍五入后的应付
} else {
    $data['amount_due'] = $totals['amount_due'];          // 用精确应付
}

$data['amount_change'] = $data['amount_due'] * -1;    // amount_due 为负表示多付

if ($data['amount_change'] > 0) {
    // 找零 > 0 表示客户多付了，需要找回
    if (array_key_exists(lang('Sales.cash'), $data['payments'])) {
        // 已有现金支付记录 → 写到该记录的 cash_refund 字段
        $data['payments'][lang('Sales.cash')]['cash_refund'] = $data['amount_change'];
    } else {
        // 没有现金支付记录 → 创建一条金额为 0 的现金记录，专门记录找零
        $payment = [
            lang('Sales.cash') => [
                'payment_type'   => lang('Sales.cash'),
                'payment_amount' => 0,
                'cash_refund'    => $data['amount_change']
            ]
        ];
        $data['payments'] += $payment;
    }
}
```

**关键设计**：找零不单独创建支付记录，而是**挂在现金支付记录上**。如果没有现金支付（比如全是刷卡支付但因某种原因需要找零），则创建一条 `payment_amount = 0` 的虚拟现金记录来承载找零。

#### 落库时的影响

**位置**：[Sale::save_value()](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Models/Sale.php#L579-L606)

```php
$total_amount = 0;         // 实际净收款金额（用于计算客户积分）
$total_amount_used = 0;    // 积分支付金额

foreach ($payments as $payment_id => $payment) {
    // 处理礼品卡：扣减礼品卡余额
    // 处理积分支付：扣减客户积分，累加到 total_amount_used

    $sales_payments_data = [
        'sale_id'         => $sale_id,
        'payment_type'    => $payment['payment_type'],
        'payment_amount'  => $payment['payment_amount'],
        'cash_refund'     => $payment['cash_refund'],       // 原样写入
        'cash_adjustment' => $payment['cash_adjustment'],   // 原样写入
        'employee_id'     => $employee_id
    ];

    $builder = $this->db->table('sales_payments');
    $builder->insert($sales_payments_data);

    // 净收款 = payment_amount - cash_refund
    $total_amount = floatval($total_amount)
        + floatval($payment['payment_amount'])
        - floatval($payment['cash_refund']);
}

// 用净收款（扣除找零后的实收）计算客户积分
$this->save_customer_rewards($customer_id, $sale_id, $total_amount, $total_amount_used);
```

**落库影响总结**：

| 维度 | cash_refund 影响 | cash_adjustment 影响 |
|------|------------------|----------------------|
| `sales_payments` 表 | 作为字段直接写入 | 作为字段直接写入 |
| 实际收款核算 | `净收款 = payment_amount - cash_refund` | 自身的 `payment_amount` 就是调整值，`cash_adjustment = 1` 标记其性质 |
| 客户积分计算 | 积分基于净收款（扣掉找零后）计算 | 调整记录会被算作一笔独立支付，但其 `payment_amount` 已纳入积分计算基数 |
| 报表统计 | 找零金额需要单独统计，不能算作收入 | 调整记录需要通过 `cash_adjustment = 1` 过滤出来单独核算 |

#### 特殊边界场景

**场景1：非现金支付产生找零**

当选择客户后应用折扣导致应付减少，而此前已用信用卡全额支付时：
- `amount_change > 0`（多付）
- 没有 `Cash` 支付记录 → 创建 `payment_amount = 0, cash_refund = 找零额` 的虚拟现金记录
- 落库后，报表需注意这条记录的现金支付为 0，只是找零记录

**场景2：现金调整方向为负（抹零）**

当 `cash_rounding_code` 配置为向下取整或舍入后金额变小时：
- `cash_adjustment_amount = 负值`
- 相当于系统给客户抹零让利
- 现金调整记录的 `payment_amount` 为负，减少整体支付总额

**场景3：现金模式下加入非现金支付**

先用现金支付（产生了现金调整），然后又加了一笔信用卡支付：
- `add_payment()` 检测到非现金支付 → `cash_mode` 置为 `FALSE`
- 已有的现金调整记录**不会被删除**
- 但 `amount_due` 计算切换为精确值（不再四舍五入）
- 结账时可能导致 `amount_change` 与之前预期的不同，产生异常找零

---

## 九、关键常量速查

定义在 [Constants.php](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Config/Constants.php)

```php
// 销售状态
const COMPLETED = 0;
const SUSPENDED = 1;
const CANCELED  = 2;

// 销售类型
const SALE_TYPE_POS        = 0;
const SALE_TYPE_INVOICE    = 1;
const SALE_TYPE_WORK_ORDER = 2;
const SALE_TYPE_QUOTE      = 3;
const SALE_TYPE_RETURN     = 4;

// 折扣类型
const PERCENT = 0;
const FIXED   = 1;

// 价格模式
const PRICE_MODE_STANDARD = 0;
const PRICE_MODE_KIT      = 1;

// 现金模式
const CASH_ADJUSTMENT_TRUE  = 1;
const CASH_ADJUSTMENT_FALSE = 0;
const CASH_MODE_TRUE        = 1;
const CASH_MODE_FALSE       = 0;

// 库存类型
const HAS_STOCK    = 0;
const HAS_NO_STOCK = 1;
```

---

## 十、核心文件索引

| 文件 | 关键方法 | 职责 |
|------|----------|------|
| [Sales.php](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Controllers/Sales.php) | `getIndex`, `postAdd`, `postAddPayment`, `postComplete`, `_reload` | 控制器，HTTP 请求处理 |
| [Sale_lib.php](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Sale_lib.php) | `add_item`, `get_totals`, `get_discount`, `add_payment`, `get_cart` | 购物车与计算逻辑 |
| [Tax_lib.php](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Libraries/Tax_lib.php) | `get_taxes`, `get_tax_for_amount`, `get_included_tax`, `round_taxes` | 税费计算 |
| [Sale.php](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Models/Sale.php) | `save_value`, `get_info`, `get_sale_items`, `get_sale_payments` | 数据库操作 |
| [Constants.php](file:///d:/fz/0601-1/solo-dogfeeding/code/11-opensourcepos/app/Config/Constants.php) | - | 常量定义 |
