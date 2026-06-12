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
| 模型 | [Sale.php](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Sale.php) | save_value, save_customer_rewards |
