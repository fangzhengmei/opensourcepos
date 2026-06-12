# 客户资料与奖励积分触发关系分析

本文档按代码执行顺序，拆解客户查询保存、积分累计、销售关联三个阶段的触发逻辑。

---

## 一、客户查询与保存

### 1.1 核心文件

| 角色 | 文件路径 |
|------|----------|
| 控制器 | [Customers.php](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Controllers/Customers.php) |
| 模型 | [Customer.php](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Customer.php) |
| 积分包模型 | [Customer_rewards.php](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Customer_rewards.php) |

### 1.2 客户数据结构

客户模型 `Customer.php` 中与积分相关的关键字段位于 `$allowedFields`：

```php
// Customer.php L18-L32
protected $allowedFields = [
    'account_number',
    'taxable',
    'tax_id',
    'sales_tax_code_id',
    'deleted',
    'discount',
    'discount_type',
    'company_name',
    'package_id',   // ★ 关联积分包ID（外键 → customers_packages.package_id）
    'points',       // ★ 客户当前累计积分余额
    'date',
    'employee_id',
    'consent'
];
```

数据库对应关系：
- `customers.package_id` → `customers_packages.package_id`（积分包）
- `customers.points` 存储积分余额

### 1.3 客户查询流程

#### 客户列表查询（getSearch）

**入口**：[Customers.php::getSearch()](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Controllers/Customers.php#L86-L117)

```
请求 AJAX → getSearch()
    ├─→ 接收 search/limit/offset/sort/order 参数
    ├─→ Customer::search()  →  联表查询 customers + people
    ├─→ Customer::get_found_rows()  →  统计总数
    └─→ 遍历每条记录：
            Customer::get_stats(person_id)  →  查询消费统计（总金额/均额/笔数）
            get_customer_data_row()         →  组装表格行数据（不含积分字段）
```

> **注意**：客户列表查询默认**不返回积分字段**，积分仅在详情页显示。

#### 客户详情查询（getView）

**入口**：[Customers.php::getView()](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Controllers/Customers.php#L146-L235)

```
点击编辑/新建 → getView(customer_id)
    ├─→ Customer::get_info(customer_id)
    │       └─→ 联表 customers + people，返回完整客户对象
    │           包含 package_id、points 等积分相关字段
    │
    ├─→ Customer::get_stats(customer_id)  →  消费统计
    │
    ├─→ ★ 加载积分包下拉选项（L173-L178）：
    │       Customer_rewards::get_all()
    │           → 查询 customers_packages 表（deleted=0）
    │           → 组装 package_id → package_name 映射
    │           → 传入视图作为下拉框选项
    │
    └─→ 传入 selected_package = info.package_id（当前客户选中的积分包）
```

#### 单个客户查询（get_info）

**模型方法**：[Customer.php::get_info()](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Customer.php#L92-L102)

```php
public function get_info(?int $person_id): object
{
    $builder = $this->db->table('customers');
    $builder->join('people', 'people.person_id = customers.person_id');
    $builder->where('customers.person_id', $person_id);
    $query = $builder->get();

    return $query->getNumRows() === 1
        ? $query->getRow()        // 返回包含 package_id、points 的完整对象
        : $this->getEmptyObject('customers');  // 空对象，points=0
}
```

### 1.4 客户保存流程

**入口**：[Customers.php::postSave()](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Controllers/Customers.php#L241-L315)

```
提交表单 → postSave(customer_id)
    ├─→ 接收并整理 person_data（姓名/邮箱/地址等基本信息）
    ├─→ 接收并整理 customer_data（L268-L280）：
    │       ├─ consent
    │       ├─ account_number
    │       ├─ tax_id / company_name
    │       ├─ discount / discount_type（客户折扣）
    │       ├─ ★ package_id（来自表单下拉框，空则设为 null）
    │       ├─ taxable / date / employee_id
    │       └─ sales_tax_code_id
    │
    └─→ Customer::save_customer(person_data, customer_data, customer_id)
            └─→ 事务内两步保存：
                ① parent::save_value(person_data) → 写入/更新 people 表
                ② 判断 NEW_ENTRY 还是已有客户：
                    - 新客户：INSERT customers 表（含 package_id 字段）
                    - 老客户：UPDATE customers 表（含 package_id 字段）
```

> **关键点**：客户保存时**只修改 `package_id` 关联，不直接操作 `points` 字段**。
> `points` 字段仅由销售流程触发变更。

### 1.5 更新积分余额方法

虽然客户保存不修改积分，但客户模型提供了专门的积分更新方法：

**方法**：[Customer.php::update_reward_points_value()](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Customer.php#L239-L244)

```php
public function update_reward_points_value(int $customer_id, int $value): void
{
    $builder = $this->db->table('customers');
    $builder->where('person_id', $customer_id);
    $builder->update(['points' => $value]);  // 直接覆盖设置为新值
}
```

> 此方法被销售模块调用，用于积分的增加（累计）和减少（抵扣支付）。

---

## 二、积分累计逻辑

### 2.1 核心数据表关系

```
┌──────────────────────┐         ┌─────────────────────────┐
│  customers_packages   │         │        customers         │
│  (积分包定义表)        │         │       (客户表)           │
├──────────────────────┤         ├─────────────────────────┤
│ package_id (PK)      │◄────────┤ package_id (FK)         │
│ package_name         │         │ person_id (PK)          │
│ points_percent       │         │ points (积分余额)        │
│ deleted              │         │ ...                     │
└──────────────────────┘         └─────────────────────────┘
                                              │
                                              │ 被消费时累计
                                              ▼
                                   ┌─────────────────────────┐
                                   │   sales_reward_points    │
                                   │   (单笔销售积分流水表)    │
                                   ├─────────────────────────┤
                                   │ id (PK, AUTO)            │
                                   │ sale_id (FK → sales)     │
                                   │ earned (本次累计积分)     │
                                   │ used (本次使用积分抵扣)   │
                                   └─────────────────────────┘
```

### 2.2 积分包模型（Customer_rewards）

**文件**：[Customer_rewards.php](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Customer_rewards.php)

对应表：`customers_packages`（客户积分等级包）

| 字段 | 说明 |
|------|------|
| package_id | 积分包ID，自增主键 |
| package_name | 积分包名称（如 "普通会员"、"VIP会员"） |
| points_percent | **积分比例**（如 5 表示每消费 100 元得 5 积分） |
| deleted | 软删除标记 |

核心方法：

```php
// 获取积分比例
public function get_points_percent(int $package_id): float
{
    $builder = $this->db->table('customers_packages');
    $builder->where('package_id', $package_id);
    return $builder->get()->getRow()->points_percent;
}

// 获取积分包名称
public function get_name(int $package_id): string
{
    $builder = $this->db->table('customers_packages');
    $builder->where('package_id', $package_id);
    return $builder->get()->getRow()->package_name;
}
```

### 2.3 积分流水模型（Rewards）

**文件**：[Rewards.php](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Rewards.php)

对应表：`sales_reward_points`（单笔销售的积分变动记录）

| 字段 | 说明 |
|------|------|
| id | 自增主键 |
| sale_id | 关联销售单号 |
| earned | 本次销售累计获得的积分 |
| used | 本次销售使用积分抵扣的金额 |

```php
// 写入积分流水（insert 或 update）
public function save_value(array &$rewards_data, bool $rewards_id = false): bool
{
    $builder = $this->db->table('sales_reward_points');
    if (!$rewards_id || !$this->exists($rewards_id)) {
        if ($builder->insert($rewards_data)) {
            $rewards_data['id'] = $this->db->insertID();
            return true;
        }
        return false;
    }
    $builder->where('id', $rewards_id);
    return $builder->update($rewards_data);
}
```

### 2.4 积分累计计算公式

**入口方法**：[Sale.php::save_customer_rewards()](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Sale.php#L1377-L1403)

```php
private function save_customer_rewards(
    int $customer_id,     // 客户ID
    int $sale_id,         // 销售单号
    float $total_amount,  // 支付总金额（实付）
    float $total_amount_used  // 本次使用积分抵扣的金额
): void
{
    $config = config(OSPOS::class)->settings;

    // ★ 前置条件1：有客户ID 且 系统开启积分开关
    if (!empty($customer_id) && $config['customer_reward_enable']) {
        $customer = model(Customer::class);
        $customer_rewards = model(Customer_rewards::class);
        $rewards = model(Rewards::class);

        // ★ 前置条件2：客户已关联积分包
        $package_id = $customer->get_info($customer_id)->package_id;

        if (!empty($package_id)) {
            // 步骤①：获取该积分包的返点比例
            $points_percent = $customer_rewards->get_points_percent($package_id);

            // 步骤②：获取客户当前积分余额（null 则视为 0）
            $points = $customer->get_info($customer_id)->points;
            $points = ($points == null ? 0 : $points);
            $points_percent = ($points_percent == null ? 0 : $points_percent);

            // 步骤③：★ 核心公式 ★
            //   本次累计积分 = 实付总金额 × 积分比例 ÷ 100
            //   例：消费 200 元，比例 5% → earned = 200 * 5 / 100 = 10 积分
            $total_amount_earned = ($total_amount * $points_percent / 100);

            // 步骤④：更新客户积分余额（累加）
            $points = $points + $total_amount_earned;
            $customer->update_reward_points_value($customer_id, $points);

            // 步骤⑤：写入积分流水（sales_reward_points 表）
            $rewards_data = [
                'sale_id' => $sale_id,
                'earned'  => $total_amount_earned,
                'used'    => $total_amount_used
            ];
            $rewards->save_value($rewards_data);
        }
    }
}
```

### 2.5 积分累计触发条件汇总

| 序号 | 条件 | 判断位置 |
|------|------|----------|
| 1 | 系统配置 `customer_reward_enable = true` | `save_customer_rewards()` L1381 |
| 2 | 销售关联了有效客户（customer_id 不为空） | L1381 |
| 3 | 该客户已分配积分包（package_id 不为空） | L1388 |
| 4 | 积分包的返点比例有效（points_percent > 0） | L1389-L1392 |

> 任何一个条件不满足，**均不会产生积分累计**。

---

## 三、销售关联逻辑

### 3.1 核心文件

| 角色 | 文件路径 |
|------|----------|
| 控制器 | [Sales.php](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Controllers/Sales.php) |
| 销售模型 | [Sale.php](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Sale.php) |

### 3.2 销售时选择客户

**入口**：[Sales.php::postSelectCustomer()](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Controllers/Sales.php#L231-L246)

```
POS 页面选择客户 → postSelectCustomer()
    ├─→ 接收 customer_id 参数
    ├─→ Customer::exists(customer_id)  →  校验客户是否存在
    │
    ├─→ sale_lib->set_customer(customer_id)  →  写入会话
    │
    ├─→ 读取客户的 discount / discount_type（客户级折扣）
    │
    ├─→ 如果有折扣：
    │       sale_lib->apply_customer_discount()  →  应用到购物车
    │
    └─→ _reload()  →  刷新 POS 页面
        └─→ 视图加载时，_load_customer_data() 会读取积分信息：
                package_id = customer->get_info()->package_id
                if (package_id != null) {
                    package_name = customer_rewards->get_name(package_id)
                    points = customer->get_info()->points
                    → 显示客户积分包名称 + 当前积分余额
                }
```

### 3.3 使用积分支付（抵扣）

**入口**：[Sales.php::postAddPayment()](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Controllers/Sales.php#L393-L479)

当用户在 POS 页面选择"奖励积分"作为支付方式时触发：

```
选择支付方式 = "Rewards（积分）" → postAddPayment()
    │
    └─→ L438-L461 积分支付分支处理：
            │
            ├─→ 步骤①：获取当前客户ID和积分包
            │       customer_id = sale_lib->get_customer()
            │       package_id  = customer->get_info(customer_id)->package_id
            │       if (!empty($package_id))  →  必须有积分包才允许积分支付
            │
            ├─→ 步骤②：获取当前积分余额
            │       points = customer->get_info(customer_id)->points ?? 0
            │
            ├─→ 步骤③：余额校验
            │       current_payments_with_rewards = 已添加的积分支付金额
            │       if (剩余积分 <= 0) {
            │           返回错误：积分不足 + 显示当前余额
            │       }
            │
            ├─→ 步骤④：计算可抵扣金额
            │       new_reward_value = points - 应付金额
            │       new_reward_value = max(new_reward_value, 0)
            │       sale_lib->set_rewards_remainder(new_reward_value)
            │       amount_tendered = min(应付金额, 积分余额)
            │
            └─→ 步骤⑤：添加积分支付记录到会话
                    sale_lib->add_payment("Rewards", amount_tendered)
```

> **重要**：此处仅将会话中的积分支付记录保存，**还未扣减数据库中的 `points`**。
> 真正扣减发生在 `Sale::save_value()` 的提交环节。

### 3.4 销售提交（保存+积分扣减+积分累计）

**入口方法**：[Sale.php::save_value()](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Sale.php#L518-L688)

这是整个流程的**核心交汇点**，按顺序执行以下步骤：

```
点击"完成销售" → Sale::save_value()
    │
    ├─► 第一阶段：保存销售主记录（L550-L574）
    │       ├─► 组装 sales_data（sale_time, customer_id, employee_id...）
    │       ├─► 校验 customer_id 是否真实存在
    │       │       customer_id = customer->exists(customer_id) ? customer_id : null
    │       ├─► 开启事务
    │       └─► INSERT/UPDATE sales 表 → 得到 sale_id
    │
    ├─► 第二阶段：遍历支付方式，扣减积分（L576-L604）
    │       │
    │       ├─► $total_amount = 0;          // 用于累计实付总额
    │       ├─► $total_amount_used = 0;     // 用于累计积分抵扣总额
    │       │
    │       └─► foreach ($payments as $payment):
    │             │
    │             ├─► 支付方式=礼品卡：扣减礼品卡余额（与积分无关）
    │             │
    │             ├─► ★ 支付方式=积分（Rewards）：L585-L589
    │             │     │
    │             │     ├─► 读取数据库中客户当前积分
    │             │     │     cur_rewards_value = customer->get_info(customer_id)->points
    │             │     │
    │             │     ├─► ★ 直接扣减数据库积分
    │             │     │     customer->update_reward_points_value(
    │             │     │         customer_id,
    │             │     │         cur_rewards_value - payment['payment_amount']
    │             │     │     )
    │             │     │
    │             │     └─► 累计：total_amount_used += payment_amount
    │             │
    │             ├─► INSERT sales_payments 表（写入支付记录）
    │             │
    │             └─► 累计：total_amount += payment_amount - cash_refund
    │
    ├─► 第三阶段：★ 触发积分累计（L606）
    │       │
    │       └─► $this->save_customer_rewards(
    │             customer_id,       // 客户ID
    │             sale_id,           // 刚保存的销售单号
    │             $total_amount,     // 实付总金额（含积分抵扣部分？注意：积分抵扣也算入了 total_amount）
    │             $total_amount_used // 本次使用积分抵扣总额
    │         )
    │         │
    │         └─► 内部执行（详见第二章 2.4 节）：
    │             ① 检查 customer_reward_enable 开关
    │             ② 检查 package_id 是否关联
    │             ③ 计算 earned = total_amount * points_percent / 100
    │             ④ UPDATE customers.points += earned
    │             ⑤ INSERT sales_reward_points (sale_id, earned, used)
    │
    ├─► 第四阶段：保存销售明细、库存、税费等（L610-L684）
    │
    └─► 提交事务 → 返回 sale_id
```

### 3.5 关于 total_amount 的注意点

在第二阶段中，积分抵扣的 `payment_amount` 也被计入了 `$total_amount`：

```php
// L603：所有支付方式（含积分抵扣）都累加
$total_amount = floatval($total_amount) + floatval($payment['payment_amount']) - floatval($payment['cash_refund']);
```

这意味着：
- 如果一笔订单用现金支付 80 元 + 积分抵扣 20 元（总应付 100 元）
- `$total_amount = 80 + 20 = 100`
- `$total_amount_used = 20`
- 积分累计 = `100 * points_percent / 100`（按订单全额返点，不扣除积分抵扣部分）

> 设计意图：积分抵扣视为一种"支付手段"，客户消费了 100 元的价值，理应按全额获得返点。

### 3.6 完整时序图

```
用户操作                        代码执行                          数据库变更
──────────────────────────────────────────────────────────────────────────
[在客户管理页面分配积分包]
  选择客户 + 积分包 → 保存
     │
     └─► Customers::postSave()
           └─► Customer::save_customer()  →  UPDATE customers.package_id

[进入 POS 收银台]
  搜索客户 → 选择客户
     │
     └─► Sales::postSelectCustomer()
           └─► session 写入 customer_id
           └─► (若有折扣) 应用客户折扣

  添加商品到购物车 → 继续...

  选择支付方式 = "奖励积分"
     │
     └─► Sales::postAddPayment()
           ├─► 校验 package_id 存在
           ├─► 校验 points 余额充足
           ├─► 计算可抵扣金额 = min(应付, 余额)
           └─► session 写入支付记录（不扣数据库）

  选择支付方式 = "现金" + 输入金额
     │
     └─► session 写入支付记录

  点击"完成销售"
     │
     └─► Sale::save_value()
           │
           ├─► ① INSERT sales (含 customer_id)
           │
           ├─► ② 遍历支付方式:
           │     ├─ 积分支付:
           │     │   UPDATE customers.points = 余额 - 抵扣额  ★扣减积分
           │     │   total_amount_used += 抵扣额
           │     └─ 现金支付:
           │         INSERT sales_payments
           │     total_amount += 各 payment_amount
           │
           ├─► ③ save_customer_rewards() ★触发累计
           │     ├─ 检查 customer_reward_enable
           │     ├─ 检查 package_id
           │     ├─ 查 points_percent
           │     ├─ earned = total_amount * points_percent / 100
           │     ├─ UPDATE customers.points += earned  ★累计积分
           │     └─ INSERT sales_reward_points
           │
           ├─► ④ INSERT sales_items（明细）
           ├─► ⑤ UPDATE inventory（库存）
           └─► 提交事务

  生成收据（显示本次积分 earned）
```

---

## 四、关键配置与开关

| 配置项 | 位置 | 说明 |
|--------|------|------|
| `customer_reward_enable` | 系统配置（OSPOS settings） | **全局开关**，为 false 时所有积分逻辑不执行 |
| `points_percent` | `customers_packages` 表 | 各积分包的返点比例（百分比数值） |
| `package_id` | `customers` 表 | 客户与积分包的关联字段，为空则不参与积分 |

---

## 五、代码优化建议

### 5.1 积分扣减顺序风险

当前顺序：**先扣减积分抵扣 → 再累计新积分**

如果客户积分余额恰好是 100，订单全额使用积分支付 100 元，返点比例 5%：
1. 扣减后 points = 0
2. 累计 earned = 100 * 5% = 5
3. 最终 points = 5（正确）

顺序逻辑正确，但建议在 `save_customer_rewards` 中增加事务保护（当前已在外层事务中）。

### 5.2 类型一致性问题

```php
// Customer_rewards.php::get_points_percent() 返回 float
// 但 customers.points 字段为 int，更新时传 int
// save_customer_rewards() L1393 计算结果为 float
$total_amount_earned = ($total_amount * $points_percent / 100);  // float
// L1394 累加到 points（int）后传入 update 方法
$points = $points + $total_amount_earned;  // int + float → float
```

建议在计算后加 `intval()` 或 `round()` 取整，避免浮点精度问题。

### 5.3 积分包下拉未处理 points 为 null 的情况

[Customers.php L174-L177](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Controllers/Customers.php#L174-L177) 直接使用 `$row['package_id']` 作为 key，建议确认表中数据完整性，或增加空值防护。

---

## 六、文件索引

| 类型 | 文件路径 | 关键方法 |
|------|----------|----------|
| 控制器 | [Customers.php](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Controllers/Customers.php) | getView, postSave, getSearch |
| 模型 | [Customer.php](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Customer.php) | get_info, save_customer, update_reward_points_value |
| 模型 | [Customer_rewards.php](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Customer_rewards.php) | get_points_percent, get_name, get_all |
| 模型 | [Rewards.php](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Rewards.php) | save_value |
| 控制器 | [Sales.php](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Controllers/Sales.php) | postSelectCustomer, postAddPayment |
| 模型 | [Sale.php](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Sale.php) | save_value, save_customer_rewards, delete, delete_suspended_sale |

---

## 七、退款退货场景中的积分回退分析

### 7.1 退货模式的两种路径

OSPOS 中有两种"退货"相关操作，对积分的影响截然不同：

| 操作 | 触发方式 | 销售类型 | 对原销售记录 | 对积分 |
|------|----------|----------|-------------|--------|
| **退货单（Return）** | POS 切换到 Return 模式 | `SALE_TYPE_RETURN = 4` | 不修改原销售 | 生成负金额退货单 |
| **删除销售（Delete）** | 管理页面删除销售 | 原记录改为 `CANCELED` | 修改 `sale_status` | **不回退积分** |

### 7.2 退货单流程（Return Mode）

#### 步骤1：进入退货模式

**入口**：[Sales.php::postChangeMode()](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Controllers/Sales.php#L254-L296)

```
mode = 'return' → sale_lib->set_sale_type(SALE_TYPE_RETURN)
```

#### 步骤2：加载原销售的商品（负数量）

**方法**：[Sale_lib.php::return_entire_sale()](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Libraries/Sale_lib.php#L1308-L1322)

```php
foreach ($this->sale->get_sale_items_ordered($sale_id)->getResult() as $row) {
    $this->add_item(
        $row->item_id,
        $row->item_location,
        -$row->quantity_purchased,  // ★ 负数量
        $row->discount,
        $row->discount_type,
        PRICE_MODE_STANDARD,
        null, null,
        $row->item_unit_price,
        $row->description,
        $row->serialnumber,
        null, true
    );
}
```

> 商品以**负数量**加入购物车，导致 `total` 为负值。

#### 步骤3：提交退货单

**入口**：[Sales.php L891-L900](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Controllers/Sales.php#L891-L900)

```php
$data['sale_status'] = COMPLETED;
if ($this->sale_lib->is_return_mode()) {
    $sale_type = SALE_TYPE_RETURN;
} else {
    $sale_type = SALE_TYPE_POS;
}
$data['sale_id_num'] = $this->sale->save_value(
    $sale_id, $data['sale_status'], $data['cart'],
    $customer_id, $employee_id, $data['comments'],
    $invoice_number, $work_order_number, $quote_number,
    $sale_type, $data['payments'], $data['dinner_table'],
    $tax_details
);
```

> 退货单仍然调用同一个 `Sale::save_value()` 方法，`sale_type = SALE_TYPE_RETURN`。

#### 步骤4：save_value 中的积分处理

在 [Sale.php::save_value()](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Sale.php#L518-L688) 中，退货单的积分处理逻辑如下：

**第二阶段（支付遍历）**：

```
foreach ($payments as $payment):
    // 退货场景：通常支付方式为 "Cash" 且 payment_amount < 0（退款给客户）
    // 如果使用积分抵扣方式支付（不太可能但理论上可以），同样会执行：
    if (payment_type == 'Rewards') {
        cur_rewards_value = customer->get_info(customer_id)->points;
        customer->update_reward_points_value(customer_id, cur_rewards_value - payment_amount);
        // 退货时 payment_amount 为负值
        // → cur_rewards_value - (-N) = cur_rewards_value + N
        // → 积分余额增加（不合理：退货币不应该增加积分余额）
    }
    total_amount += payment_amount - cash_refund;
    // 退货时 payment_amount < 0
    // → total_amount 为负值
```

**第三阶段（save_customer_rewards）**：

```
save_customer_rewards(customer_id, sale_id, total_amount, total_amount_used):
    // total_amount < 0（退货金额为负）
    points_percent = customer_rewards->get_points_percent(package_id);
    total_amount_earned = total_amount * points_percent / 100;
    // ★ 例如：total_amount = -100, points_percent = 5
    //   earned = -100 * 5 / 100 = -5
    // → earned 为负值
    
    points = points + total_amount_earned;
    // → 积分余额减少（回退原销售累计的积分）
    
    customer->update_reward_points_value(customer_id, points);
    // → 写入负的 earned 到 sales_reward_points 流水
```

### 7.3 退货场景积分变动的完整推演

假设原销售：
- 消费 100 元，积分比例 5%，使用现金支付
- 原销售后：earned = 5，points 从 0 → 5

退货操作：
- 退货金额 = -100 元
- total_amount = -100（负值支付金额）

| 阶段 | 计算 | 结果 |
|------|------|------|
| save_customer_rewards | earned = -100 × 5 / 100 = -5 | earned = **-5** |
| 更新 points | points = 5 + (-5) = 0 | points = **0** |
| 写入流水 | sales_reward_points: earned = -5, used = 0 | 记录负值流水 |

**结论**：退货时 `save_customer_rewards` **能够回退原销售累计的积分**，因为 `total_amount` 为负值导致 `earned` 也为负值，进而从 `points` 中扣减。

### 7.4 删除销售场景（Delete Sale）

**方法**：[Sale.php::delete()](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Sale.php#L795-L839)

```php
public function delete($sale_id = null, bool $purge = false, bool $update_inventory = true, $employee_id = null): bool
{
    $this->db->transStart();

    $sale_status = $this->get_sale_status($sale_id);

    if ($update_inventory && $sale_status == COMPLETED) {
        // ★ 只更新了库存，没有回退积分！
        // ... 库存恢复逻辑 ...
    }

    $this->update_sale_status($sale_id, CANCELED);

    $this->db->transComplete();
    return $this->db->transStatus();
}
```

**严重问题**：`Sale::delete()` 方法**只更新库存和状态，完全没有处理积分回退**。

| 操作 | 库存恢复 | 积分回退 | 积分流水清理 |
|------|---------|---------|-------------|
| 退货单（Return） | ✅ 负数量恢复 | ✅ earned 为负自动回退 | ✅ 新增负值流水 |
| 删除销售（Delete） | ✅ 显式恢复 | ❌ **不回退** | ❌ **不清理** |
| 删除挂起销售 | ❌ 不恢复 | ❌ 不回退 | ❌ 不清理 |

删除已完成的销售后：
- `customers.points` 中仍保留该销售累计的积分 → **积分虚增**
- `sales_reward_points` 中仍保留该销售的 earned 记录 → **流水不一致**
- 如果该销售使用了积分支付，已扣减的积分也不会恢复 → **积分净损失**

### 7.5 退货时积分支付方式的潜在问题

如果在退货操作中用户选择"积分"作为退款方式（即将积分退回客户账户），在支付遍历阶段：

```php
// Sale.php L585-L589
elseif (!empty(strstr($payment['payment_type'], lang('Sales.rewards')))) {
    $cur_rewards_value = $customer->get_info($customer_id)->points;
    $customer->update_reward_points_value($customer_id, $cur_rewards_value - $payment['payment_amount']);
    // 退货时 payment_amount 为负值
    // cur_rewards_value - (-N) = cur_rewards_value + N → 积分余额增加
    $total_amount_used = floatval($total_amount_used) + floatval($payment['payment_amount']);
    // total_amount_used 也会变成负值
}
```

然后在 `save_customer_rewards` 中：
```
earned = total_amount × points_percent / 100  （负值）
```

这意味着退货如果用积分"支付"（实为退回积分），客户会得到**双重积分恢复**：
1. 支付阶段：points += |payment_amount|（积分退回）
2. 累计阶段：points += earned（earned 为负，等价于减少）

但逻辑上只有支付阶段的退回是合理的，累计阶段的负 earned 会额外扣减，造成**积分净损失**。

---

## 八、多终端操作积分余额的竞态风险分析

### 8.1 当前积分更新的读-改-写模式

积分余额的更新采用**先读后写**模式，未使用数据库级锁：

**扣减积分（支付时）**：[Sale.php L585-L589](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Sale.php#L585-L589)

```php
// 步骤1：读取当前余额（SELECT）
$cur_rewards_value = $customer->get_info($customer_id)->points;

// 步骤2：PHP 层计算新值
$new_value = $cur_rewards_value - $payment['payment_amount'];

// 步骤3：写入新值（UPDATE）
$customer->update_reward_points_value($customer_id, $new_value);
// → UPDATE customers SET points = $new_value WHERE person_id = $customer_id
```

**累计积分（save_customer_rewards）**：[Sale.php L1389-L1396](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Sale.php#L1389-L1396)

```php
// 步骤1：读取当前余额（SELECT）
$points = $customer->get_info($customer_id)->points;

// 步骤2：PHP 层累加
$points = $points + $total_amount_earned;

// 步骤3：写入新值（UPDATE）
$customer->update_reward_points_value($customer_id, $points);
```

### 8.2 竞态条件场景推演

**场景**：两个终端同时对同一客户进行销售

```
时间线   终端A（消费100元，积分支付20元）         终端B（消费80元）
─────── ─────────────────────────────── ──────────────────────────
T1      读取 points = 100
T2                                           读取 points = 100
T3      计算: 100 - 20 = 80
T4      UPDATE points = 80
T5                                           计算: 100 + 80×5% = 104
T6                                           UPDATE points = 104
─────── ─────────────────────────────── ──────────────────────────
结果    points = 104（应该是 100 - 20 + 4 = 84）
        终端A的积分扣减被覆盖 → 积分虚增 20
```

**根因**：`update_reward_points_value()` 使用的是**绝对值覆盖**而非**增量更新**：

```php
// Customer.php L239-L244
public function update_reward_points_value(int $customer_id, int $value): void
{
    $builder = $this->db->table('customers');
    $builder->where('person_id', $customer_id);
    $builder->update(['points' => $value]);  // ★ 覆盖写入，非增量
}
```

### 8.3 风险等级评估

| 因素 | 评估 |
|------|------|
| 发生概率 | 低~中（需要同一客户在两个终端同时结账） |
| 影响程度 | 高（积分余额错误，可能导致积分被盗用） |
| 事务保护 | 部分有效（同一事务内的回滚能保护，但跨事务无隔离） |
| 锁机制 | **无**（未使用 `SELECT ... FOR UPDATE` 或 `UPDATE ... SET points = points ± N`） |

### 8.4 建议修复方案

**方案A：改用 SQL 增量更新（推荐）**

```php
// 扣减积分
$builder = $this->db->table('customers');
$builder->where('person_id', $customer_id);
$builder->set('points', 'points - ' . (int)$payment_amount, false);
$builder->update();

// 累计积分
$builder = $this->db->table('customers');
$builder->where('person_id', $customer_id);
$builder->set('points', 'points + ' . (float)$total_amount_earned, false);
$builder->update();
```

优点：原子操作，天然避免竞态；缺点：需修改 `update_reward_points_value` 方法签名。

**方案B：加行级锁**

```php
// 在事务开始后立即加锁
$builder = $this->db->table('customers');
$builder->select('points');
$builder->where('person_id', $customer_id);
$builder->forUpdate();  // SELECT ... FOR UPDATE
$points = $builder->get()->getRow()->points;
```

优点：保持现有方法结构；缺点：需要在事务中显式加锁。

**方案C：乐观锁（版本号）**

在 `customers` 表增加 `version` 字段，UPDATE 时带版本条件。

---

## 九、浮点计算精度对积分累计的实际影响分析

### 9.1 类型链路分析

积分计算涉及三个层级的类型转换：

```
数据库字段类型          PHP 代码类型           计算过程
─────────────────    ─────────────────    ──────────────
customers.points     → int (PHP)           $points: int
    int(11)               ↓
                    Customer::get_info()
                    → $info->points: int|null
                         ↓
                    $points = ($points == null ? 0 : $points)  → int
                         ↓
customers_packages   → float (PHP)         $points_percent: float
.points_percent          ↓
    float            Customer_rewards::get_points_percent()
                    → return float
                         ↓
                    $points_percent = ($points_percent == null ? 0 : $points_percent) → float
                         ↓
                    ★ 核心计算 ★
                    $total_amount_earned = ($total_amount * $points_percent / 100);
                    // float * float / int → float
                         ↓
                    $points = $points + $total_amount_earned;
                    // int + float → float  ★ 类型升级
                         ↓
                    $customer->update_reward_points_value($customer_id, $points);
                    // 方法签名: update_reward_points_value(int $customer_id, int $value)
                    // ★ float → int 隐式转换，PHP 8.1+ 会产生 TypeError
```

### 9.2 PHP 8.1+ 严格类型下的致命问题

[Customer.php L239](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Customer.php#L239)：

```php
public function update_reward_points_value(int $customer_id, int $value): void
```

方法签名要求 `$value` 为 `int`，但在 `save_customer_rewards` 中：

```php
// Sale.php L1393-L1396
$total_amount_earned = ($total_amount * $points_percent / 100);  // float
$points = $points + $total_amount_earned;                         // float
$customer->update_reward_points_value($customer_id, $points);     // ★ float → int
```

在 PHP 8.1+ 严格模式下，**将 float 传入 `int` 类型参数会触发 `TypeError`**，导致积分更新失败。

即使在非严格模式下（隐式截断），也会发生**精度丢失**：

| 示例 | 计算过程 | 期望值 | 实际值（隐式截断） | 误差 |
|------|----------|--------|-------------------|------|
| 消费 99.99 元，比例 5% | 99.99 × 5 / 100 = 4.9995 | 5 | 4 | -1 |
| 消费 33.33 元，比例 10% | 33.33 × 10 / 100 = 3.333 | 3 | 3 | -0.333 |
| 消费 0.01 元，比例 5% | 0.01 × 5 / 100 = 0.0005 | 0 | 0 | 精度丢失 |

### 9.3 浮点存储的精度风险

数据库 `sales_reward_points` 表的字段类型：

```sql
-- 3.0.2_to_3.1.1.sql L111-L117
CREATE TABLE `ospos_sales_reward_points` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `sale_id` int(11) NOT NULL,
  `earned` float NOT NULL,    -- ★ float 类型
  `used` float NOT NULL,      -- ★ float 类型
  PRIMARY KEY (`id`)
);
```

`float` 类型在 MySQL 中为单精度浮点数（4字节，约7位有效数字），存在精度丢失：

```
例如：earned = 123456.78
float 存储 → 123456.77（末位偏差）
多次累计后误差会逐步放大
```

`customers_packages.points_percent` 也是 `float`：

```sql
`points_percent` float NOT NULL DEFAULT '0',
```

同样存在精度问题，例如设置 `points_percent = 33.33`，实际存储可能为 `33.329998`。

### 9.4 精度问题在 long-running 系统中的累积效应

考虑一个高频消费场景（每天 100 笔交易，每笔平均 50 元，积分比例 5%）：

| 时间跨度 | 理论累计积分 | 浮点累计误差（估） | 误差率 |
|----------|-------------|-------------------|--------|
| 1 天 | 250 | ~0.01 | 0.004% |
| 1 月 | 7,500 | ~0.3 | 0.004% |
| 1 年 | 91,250 | ~3.65 | 0.004% |

单次误差极小，但长期运行后在 `customers.points`（int 类型）与 `SUM(sales_reward_points.earned)`（float 类型）之间会出现**不可解释的差额**。

### 9.5 修复建议

| 问题 | 修复方案 |
|------|----------|
| PHP float → int TypeError | 在 `save_customer_rewards` 中调用前 `round()` 或 `intval()` |
| `sales_reward_points.earned` float 精度 | 改为 `DECIMAL(15,4)` |
| `customers_packages.points_percent` float 精度 | 改为 `DECIMAL(5,2)` |
| `customers.points` int 与 float 计算不匹配 | 统一使用 `DECIMAL` 或全部 `intval()` 取整 |

---

## 十、积分更新失败后的事务回滚分析

### 10.1 事务边界梳理

`Sale::save_value()` 的事务结构：

```php
// Sale.php L564-L687
$this->db->transStart();        // ★ 事务开始

// ① INSERT/UPDATE sales 表
// ② 遍历支付：
//      - 礼品卡扣减
//      - ★ 积分扣减：update_reward_points_value()   ← 在事务内
//      - INSERT sales_payments
// ③ ★ 积分累计：save_customer_rewards()             ← 在事务内
//      - update_reward_points_value()                 ← 在事务内
//      - rewards->save_value()                        ← 在事务内
// ④ 遍历商品：INSERT sales_items + 库存更新
// ⑤ 税费保存
// ⑥ 餐桌状态更新

$this->db->transComplete();     // ★ 事务结束
return $this->db->transStatus() ? $sale_id : -1;
```

### 10.2 CodeIgniter 4 事务机制分析

`transStart()` / `transComplete()` 的工作原理：

```
transStart():
    - 禁用自动提交
    - 开始事务

transComplete():
    - 检查是否有查询失败（transStatus 标志）
    - 如果失败 → ROLLBACK
    - 如果成功 → COMMIT
    - 恢复自动提交
```

关键点：CodeIgniter 4 通过**查询结果标志**来判断是否回滚，而非异常捕获。

### 10.3 各积分操作的失败检测

| 操作 | 返回值 | 失败检测方式 | 是否影响 transStatus |
|------|--------|-------------|---------------------|
| `$builder->insert()` | bool | 返回 false 时设置标志 | ✅ 是 |
| `$builder->update()` | bool | 返回 false 时设置标志 | ✅ 是 |
| `update_reward_points_value()` | **void** | ★ **不返回任何值** | ❌ **否** |
| `rewards->save_value()` | bool | 返回 false 时设置标志 | ✅ 是 |

### 10.4 致命问题：update_reward_points_value 无返回值

[Customer.php L239-L244](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Customer.php#L239-L244)：

```php
public function update_reward_points_value(int $customer_id, int $value): void
{
    $builder = $this->db->table('customers');
    $builder->where('person_id', $customer_id);
    $builder->update(['points' => $value]);
    // ★ 返回类型为 void，不检查 $builder->update() 的结果
    // ★ 即使 UPDATE 失败，也不会设置 transStatus 为 false
}
```

**影响链路**：

1. **积分扣减失败**（L587）→ `update_reward_points_value` 返回 void → `transStatus` 不受影响 → **事务不回滚** → 销售记录已保存，但积分未被扣减 → **积分虚增**
2. **积分累计失败**（L1396）→ 同上 → **事务不回滚** → 销售记录已保存，但积分未累计 → **积分少计**
3. **积分流水写入失败**（L1400）→ `rewards->save_value()` 返回 bool → 如果返回 false，CodeIgniter 会检测到 → **事务回滚** ✅

### 10.5 嵌套事务问题

`save_value()` 中调用了 `clear_suspended_sale_detail()`（L543），该方法内部也有自己的事务：

```php
// Sale.php L1330-L1356
public function clear_suspended_sale_detail(int $sale_id): bool
{
    $this->db->transStart();    // ★ 嵌套事务
    // ... 删除 payments, items, taxes ...
    $this->db->transComplete();
    return $this->db->transStatus();
}
```

CodeIgniter 4 支持嵌套事务（通过 savepoint），但 `clear_suspended_sale_detail` 的返回值在 `save_value` 中**未被检查**：

```php
// Sale.php L542-L544
if ($sale_id != NEW_ENTRY) {
    $this->clear_suspended_sale_detail($sale_id);
    // ★ 返回值被忽略！如果清空失败，后续仍会继续写入
}
```

### 10.6 事务回滚完整性矩阵

| 失败环节 | 是否触发回滚 | 回滚范围 | 积分一致性影响 |
|----------|-------------|---------|---------------|
| INSERT sales 失败 | ✅ | 整个事务 | 无影响（全部回滚） |
| 积分扣减 `update_reward_points_value` 失败 | ❌ | **不回滚** | **积分虚增**（销售已保存但积分未扣减） |
| 积分累计 `update_reward_points_value` 失败 | ❌ | **不回滚** | **积分少计**（销售已保存但积分未累计） |
| 积分流水 `rewards->save_value` 失败 | ✅ | 整个事务 | 无影响（全部回滚） |
| INSERT sales_items 失败 | ✅ | 整个事务 | 无影响（全部回滚） |
| `clear_suspended_sale_detail` 失败 | ❌ | 仅子事务 | 销售明细重复/混乱 |

### 10.7 修复建议

**1. `update_reward_points_value` 改为返回 bool**

```php
public function update_reward_points_value(int $customer_id, int $value): bool
{
    $builder = $this->db->table('customers');
    $builder->where('person_id', $customer_id);
    return $builder->update(['points' => $value]) !== false;
}
```

**2. 在 save_value 中检查返回值**

```php
// 积分扣减
if (!$customer->update_reward_points_value($customer_id, $cur_rewards_value - $payment['payment_amount'])) {
    $this->db->transRollback();
    return -1;
}
```

**3. 检查 clear_suspended_sale_detail 返回值**

```php
if ($sale_id != NEW_ENTRY) {
    if (!$this->clear_suspended_sale_detail($sale_id)) {
        $this->db->transRollback();
        return -1;
    }
}
```

---

## 十一、综合风险评估与优先级建议

| 风险 | 严重程度 | 发生概率 | 修复优先级 | 修复难度 |
|------|---------|---------|-----------|---------|
| 删除销售不回退积分 | **严重** | 高（每次删除都触发） | P0 | 中 |
| `update_reward_points_value` 返回 void 导致静默失败 | **严重** | 低（DB异常时触发） | P0 | 低 |
| PHP 8.1+ float→int TypeError | **严重** | 高（每次积分累计都触发） | P0 | 低 |
| 多终端竞态条件 | 中 | 低 | P1 | 中 |
| 浮点精度累积误差 | 低 | 确定（长期运行） | P2 | 中 |
| 退货+积分支付双重回退 | 中 | 低 | P1 | 高 |
