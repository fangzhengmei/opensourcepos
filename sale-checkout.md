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

优先级：商品套装折扣 > 客户折扣 > 系统默认折扣

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

---

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
