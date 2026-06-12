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
    │             ├─► 支付方式=礼品卡：扣减礼品卡余额（⚠️ 金额会计入 total_amount 参与积分累计）
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

### 3.6 礼品卡支付对积分累计的影响

#### 3.6.1 代码定位

在 [Sale.php::save_value()](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Sale.php#L579-L606) 的支付遍历逻辑中，礼品卡支付的金额被全额计入 `$total_amount`：

```php
foreach ($payments as $payment_id => $payment) {
    // L580-L584：礼品卡支付 → 扣减礼品卡余额
    if (!empty(strstr($payment['payment_type'], lang('Sales.giftcard')))) {
        $splitpayment = explode(':', $payment['payment_type']);
        $cur_giftcard_value = $giftcard->get_giftcard_value($splitpayment[1]);
        $giftcard->update_giftcard_value($splitpayment[1], $cur_giftcard_value - $payment['payment_amount']);
    }
    // L585-L589：积分支付 → 扣减积分余额 + 累加 total_amount_used
    elseif (!empty(strstr($payment['payment_type'], lang('Sales.rewards')))) {
        // ...
    }

    // L591-L601：写入 sales_payments

    // L603：★ 所有支付方式统一累加，含礼品卡、积分、现金
    $total_amount = floatval($total_amount) + floatval($payment['payment_amount']) - floatval($payment['cash_refund']);
}

// L606：用累加后的 total_amount 计算积分
$this->save_customer_rewards($customer_id, $sale_id, $total_amount, $total_amount_used);
```

**核心发现**：礼品卡支付的 `payment_amount` 被全额计入 `$total_amount`，进而在 `save_customer_rewards` 中参与了积分累计计算。

#### 3.6.2 礼品卡 vs 积分支付的处理对比

| 维度 | 积分支付（Rewards） | 礼品卡支付（Giftcard） | 是否计入积分累计 |
|------|-------------------|---------------------|----------------|
| 支付时扣减余额 | ✅ 扣减 `customers.points` | ✅ 扣减 `giftcards.value` | — |
| `total_amount_used` 累加 | ✅ 累加 | ❌ 不累加 | — |
| `total_amount` 累加 | ✅ 累加 | ✅ 累加 | — |
| 最终积分累计计算 | ✅ 参与 earned 计算 | ✅ **参与 earned 计算** | ✅ |

#### 3.6.3 业务场景推演

**场景：现金 120 元 + 礼品卡 80 元，合计 200 元订单，积分比例 5%**

当前计算：
```
total_amount = 80（礼品卡） + 120（现金） = 200
earned = 200 × 5% = 10 积分
```

如果按"仅实付现金返积分"策略：
```
total_amount = 120（仅现金部分）
earned = 120 × 5% = 6 积分
```

两种策略相差 4 积分，差异率 40%。

#### 3.6.4 合理性分析

礼品卡的积分返点是否合理，取决于礼品卡的发放来源：

| 发放场景 | 发放时是否有现金流入 | 消费时返积分是否合理 |
|---------|-------------------|-------------------|
| 客户现金购买礼品卡 | ✅ 有（购卡时支付） | ✅ 合理（等价于现金支付） |
| 营销活动赠送礼品卡 | ❌ 无（成本计入营销费用） | ⚠️ 存疑（相当于无成本获积分） |
| 第三方/企业福利卡 | ⚠️ 结算时才有 | 取决于商务约定 |

> **注意**：当前 [Giftcards.php 控制器](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Controllers/Giftcards.php) 在发售礼品卡时**未触发任何积分累计**。如果采用"礼品卡支付不返积分"的策略，需同步在购卡环节增加积分累计，否则客户在"购卡→消费"两个环节都拿不到积分。

更多深度分析和修复方案见第十二章。

### 3.7 完整时序图

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

> **注意**：目前系统无单独的"礼品卡是否参与积分累计"配置开关。礼品卡支付金额会被全额计入 `total_amount` 参与积分计算（详见第三章 3.6 节和第十二章）。

---

## 五、代码优化建议

### 5.1 积分扣减顺序风险

当前顺序：**先扣减积分抵扣 → 再累计新积分**

如果客户积分余额恰好是 100，订单全额使用积分支付 100 元，返点比例 5%：
1. 扣减后 points = 0
2. 累计 earned = 100 * 5% = 5
3. 最终 points = 5（正确）

顺序逻辑正确，但建议在 `save_customer_rewards` 中增加事务保护（当前已在外层事务中）。

### 5.2 类型一致性与强制截断问题

```php
// Customer_rewards.php::get_points_percent() 返回 float
// customers.points 字段为 int，更新方法参数声明为 int
// save_customer_rewards() L1393 计算结果为 float
$total_amount_earned = ($total_amount * $points_percent / 100);  // float
// L1394 累加到 points（int）后传入 update 方法
$points = $points + $total_amount_earned;  // int + float → float
```

PHP 在非严格模式下会对 `float → int` 执行**向零截断取整**，可能导致每笔交易少计 0~1 积分（详见第九章 9.2 节）。建议在计算后加 `intval()` 或 `round()` 取整，显式控制精度策略。

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
                    // ★ float → int 隐式转换，PHP 非严格模式下做强制截断
```

### 9.2 PHP 非严格模式下的强制截断影响

经核实，项目 `app/` 目录下**所有文件均未声明 `declare(strict_types=1)`**，因此不存在 PHP 8.1+ 的 TypeError 风险。在 PHP 默认模式下，将 `float` 传入声明为 `int` 的参数会执行**强制类型转换（向零截断取整）**。

[Customer.php L239](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Customer.php#L239)：

```php
public function update_reward_points_value(int $customer_id, int $value): void
```

方法签名要求 `$value` 为 `int`，但在 `save_customer_rewards` 中：

```php
// Sale.php L1393-L1396
$total_amount_earned = ($total_amount * $points_percent / 100);  // float
$points = $points + $total_amount_earned;                         // float
$customer->update_reward_points_value($customer_id, $points);     // float → (int)强制截断
```

PHP 执行 `(int)` 转换时采用**向零截断策略**（`intval()` 等价行为）：

| 示例 | 计算过程 | 四舍五入期望值 | PHP 实际 (int) 值 | 误差 |
|------|----------|--------------|------------------|------|
| 消费 99.99 元，比例 5% | 99.99 × 5 / 100 = 4.9995 | 5 | 4 | **-1 积分** |
| 消费 33.33 元，比例 10% | 33.33 × 10 / 100 = 3.333 | 3 | 3 | -0.333（截断在整数位无损失） |
| 消费 0.01 元，比例 5% | 0.01 × 5 / 100 = 0.0005 | 0 | 0 | 精度丢失（小数部分） |
| 消费 19.99 元，比例 5% | 19.99 × 5 / 100 = 0.9995 | 1 | **0** | **-1 积分** |
| 消费 299.99 元，比例 5% | 299.99 × 5 / 100 = 14.9995 | 15 | **14** | **-1 积分** |

#### 实际业务影响评估

- **高频小金额消费场景**（如便利店客单价 20~30 元）：每 1~2 笔就会产生 1 积分的截断损失
- **正常零售场景**（客单价 50~200 元，比例 5%）：约每 2~10 笔损失 1 积分
- **大额消费场景**（客单价 > 1000 元）：因 earned 数值大，小数部分占比极小，截断损失可忽略
- **客户感知**：单客每月少 2~10 积分，长期累积客户可感知积分余额与理论值不符，但单笔差异极小不易察觉

#### PHP 强制截断 vs 数据库 float 截断的双重风险

除了 PHP 层的 `(int)` 截断，MySQL `customers.points` 字段为 `INT(11)`，写入时也会再次执行整数化转换。两层截断如果策略不一致（PHP 向零截断 vs MySQL 根据 SQL 模式可能 ROUND），可能造成额外的 0.5 积分级别偏差。

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
| PHP float → int 强制截断（向零取整） | 在 `save_customer_rewards` 中调用前 `round()` 或 `intval()` |
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
//      - 礼品卡扣减（update_giftcard_value）
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

### 10.2 CodeIgniter 4 事务内部追踪机制

#### 10.2.1 CI4 BaseConnection 的事务状态追踪原理

CodeIgniter 4 的 `BaseConnection` 类维护一个**内部状态标志 `$_trans_status`**（布尔值，默认 `true`）。每次执行查询时，无论应用层是否接收返回值，连接对象都会自动追踪查询执行结果：

```
应用层调用 $builder->update(...) / insert(...) / delete(...)
    │
    └─→ BaseConnection::query() 底层执行 SQL
            │
            ├─→ 成功 → $_trans_status 保持当前值（不改变）
            │
            └─→ 失败（SQL 语法错误、主键冲突、锁超时、字段类型不匹配等）
                    └─→ $_trans_status = false  ★ 内部自动标记失败
                            │
                            └─→ 与应用层方法的返回值类型无关
                                即使方法返回 void，标志已写入连接对象

应用层调用 $this->db->transComplete()
    │
    └─→ 检查内部 $_trans_status 标志
            ├─→ true  → COMMIT
            └─→ false → ROLLBACK  ★ 自动回滚整个事务
```

#### 10.2.2 项目中事务使用模式验证

从项目代码中观察到一致的使用模式（以 [Supplier.php](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Supplier.php#L113-L128) 为例）：

```php
$this->db->transStart();
// ... 一系列 insert/update ...
$this->db->transComplete();
$success &= $this->db->transStatus();  // ★ 统一通过 transStatus() 检查
```

多个模型（Tax_jurisdiction、Tax_category、Tax_code、Receiving 等）均采用相同模式，**从未检查单个 update/insert 的返回值**，全部依赖 `transStatus()` 最终检查。这证明 CI4 内部追踪机制是项目默认依赖的设计。

### 10.3 各积分操作的失败检测与回滚验证

| 操作 | 应用层返回值 | 底层调用 `$builder->update()` | CI4 内部是否标记 `$_trans_status=false` | 事务回滚 |
|------|------------|----------------------------|---------------------------------------|----------|
| `$builder->insert()` | bool | 是 | ✅ 是 | ✅ 是 |
| `$builder->update()` | bool | 是 | ✅ 是 | ✅ 是 |
| `update_reward_points_value()` | **void** | 是（内部调用 `$builder->update()`） | ✅ 是（CI4 内部追踪） | ✅ 是 |
| `update_giftcard_value()` | **void** | 是（内部调用 `$builder->update()`） | ✅ 是（CI4 内部追踪） | ✅ 是 |
| `rewards->save_value()` | bool | 是 | ✅ 是 | ✅ 是 |

### 10.4 `update_reward_points_value` 的真实行为

[Customer.php L239-L244](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Customer.php#L239-L244)：

```php
public function update_reward_points_value(int $customer_id, int $value): void
{
    $builder = $this->db->table('customers');
    $builder->where('person_id', $customer_id);
    $builder->update(['points' => $value]);
    // 返回值被丢弃，但 CI4 连接层内部已将执行结果写入 $_trans_status
    // 如果 update() 返回 false，BaseConnection::query() 已自动设置 $_trans_status = false
}
```

**关键点**：`$builder->update()` 返回 `bool`，但该返回值被方法忽略。然而，在 `$builder->update()` 调用 `BaseConnection::query()` 执行 SQL 时，**连接对象内部已经根据执行结果更新了 `$_trans_status`**，与应用层是否接收返回值无关。

### 10.5 典型失败场景的回滚验证

假设以下失败场景发生在事务内，验证是否能正确回滚：

| 失败场景 | 发生位置 | `$_trans_status` 是否为 false | 事务结果 | 积分一致性 |
|---------|---------|----------------------------|----------|-----------|
| WHERE 条件匹配 0 行（客户被删除） | L587 积分扣减 | ✅ 是（affected_rows=0 时 CI4 视为成功，但逻辑上失败） | COMMIT ⚠️ | 积分未扣减但销售已保存 |
| SQL 语法错误 / 字段类型错误 | 任意 update | ✅ 是 | ROLLBACK ✅ | 无影响 |
| 锁等待超时（InnoDB lock wait timeout） | 任意 update | ✅ 是 | ROLLBACK ✅ | 无影响 |
| 数据库连接断开 | 任意查询 | ✅ 是 | ROLLBACK ✅ | 无影响 |
| 积分流水 `rewards->save_value` 失败 | L1400 | ✅ 是 | ROLLBACK ✅ | 无影响 |

**注意**：`affected_rows=0` 的情况（如 WHERE 条件匹配不到行）CI4 视为成功返回，但从业务逻辑看属于失败。这种场景下 `$_trans_status` 仍为 `true`，事务会 COMMIT，导致**积分未扣减但销售已保存**。这是一个逻辑层面的问题，不是 CI4 事务机制的问题。

### 10.6 嵌套事务问题

`save_value()` 中调用了 `clear_suspended_sale_detail()`（L543），该方法内部也有自己的事务：

```php
// Sale.php L1330-L1356
public function clear_suspended_sale_detail(int $sale_id): bool
{
    $this->db->transStart();    // ★ 嵌套事务（SAVEPOINT）
    // ... 删除 payments, items, taxes ...
    $this->db->transComplete();
    return $this->db->transStatus();
}
```

CodeIgniter 4 支持嵌套事务（通过 SAVEPOINT），但 `clear_suspended_sale_detail` 的返回值在 `save_value` 中**未被检查**：

```php
// Sale.php L542-L544
if ($sale_id != NEW_ENTRY) {
    $this->clear_suspended_sale_detail($sale_id);
    // ★ 返回值被忽略！子事务失败时只会回滚到 savepoint，外层事务继续执行
}
```

### 10.7 事务回滚完整性矩阵（最终修正版）

| 失败环节 | 是否触发回滚 | 回滚范围 | 积分一致性影响 |
|----------|-------------|---------|---------------|
| INSERT sales 失败 | ✅ | 整个事务 | 无影响（全部回滚） |
| 积分扣减 `update_reward_points_value` SQL 执行失败 | ✅ | 整个事务（CI4 内部机制） | 无影响（全部回滚） |
| 积分扣减 WHERE 匹配 0 行（逻辑失败） | ❌ | 不回滚（CI4 视为成功） | **积分虚增**（销售已保存但积分未扣减） |
| 积分累计 `update_reward_points_value` SQL 执行失败 | ✅ | 整个事务（CI4 内部机制） | 无影响（全部回滚） |
| 积分累计 WHERE 匹配 0 行（逻辑失败） | ❌ | 不回滚（CI4 视为成功） | **积分少计**（销售已保存但积分未累计） |
| 礼品卡扣减 `update_giftcard_value` 失败 | ✅ | 整个事务（CI4 内部机制） | 无影响（全部回滚） |
| 积分流水 `rewards->save_value` 失败 | ✅ | 整个事务 | 无影响（全部回滚） |
| INSERT sales_items 失败 | ✅ | 整个事务 | 无影响（全部回滚） |
| `clear_suspended_sale_detail` 子事务失败 | ⚠️ 部分 | 仅子事务回滚到 savepoint | 可能造成销售明细重复 |

### 10.8 修复建议

**1. 增加受影响行数校验（解决 WHERE 匹配 0 行问题）**

```php
public function update_reward_points_value(int $customer_id, int $value): bool
{
    $builder = $this->db->table('customers');
    $builder->where('person_id', $customer_id);
    $builder->update(['points' => $value]);
    
    // 检查受影响行数，0 行也视为失败
    return $this->db->affectedRows() > 0;
}
```

**2. 在 save_value 中检查返回值并显式回滚**

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

**4. `update_giftcard_value` 同样需要增加返回值（与积分对称）**

[Giftcard.php L307-L312](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Giftcard.php#L307-L312) 当前也是 `void`，与积分存在相同的 `affected_rows=0` 逻辑失败问题。

---

## 十一、综合风险评估与优先级建议

| 风险 | 严重程度 | 发生概率 | 修复优先级 | 修复难度 |
|------|---------|---------|-----------|---------|
| 删除销售不回退积分 | **严重** | 高（每次删除都触发） | P0 | 中 |
| 积分扣减/累计 WHERE 匹配 0 行（逻辑失败，CI4 视为成功不回滚） | **严重** | 中（客户数据异常时触发） | P0 | 低 |
| PHP 强制截断积分（float→int 向零取整） | 中 | **高（每笔交易都触发）** | P1 | 低 |
| 礼品卡支付金额计入积分累计（设计缺陷） | 中 | 高（每笔礼品卡支付都触发） | P1 | 中 |
| 多终端竞态条件（先读后写覆盖） | 中 | 低（需并发结账） | P1 | 中 |
| 退货+积分支付双重回退逻辑 | 中 | 低（退货场景少见） | P1 | 高 |
| `update_reward_points_value` 返回 void（仅影响可观测性，不影响事务回滚） | 低 | 低（仅调试时不便） | P2 | 低 |
| `update_giftcard_value` 返回 void（同上） | 低 | 低 | P2 | 低 |
| 浮点精度累积误差（MySQL float 存储） | 低 | 确定（长期运行缓慢累积） | P2 | 中 |
| `clear_suspended_sale_detail` 返回值被忽略 | 低 | 低（仅编辑挂起单时触发） | P2 | 低 |

---

## 十二、礼品卡支付计入积分累计的设计问题分析

### 12.1 问题定位

在 [Sale.php::save_value()](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Sale.php#L579-L606) 的支付遍历逻辑中：

```php
foreach ($payments as $payment_id => $payment) {
    // L580-L584：礼品卡支付 → 扣减礼品卡余额
    if (!empty(strstr($payment['payment_type'], lang('Sales.giftcard')))) {
        $splitpayment = explode(':', $payment['payment_type']);
        $cur_giftcard_value = $giftcard->get_giftcard_value($splitpayment[1]);
        $giftcard->update_giftcard_value($splitpayment[1], $cur_giftcard_value - $payment['payment_amount']);
    }
    // L585-L589：积分支付 → 扣减积分余额 + 累加 total_amount_used
    elseif (!empty(strstr($payment['payment_type'], lang('Sales.rewards')))) {
        $cur_rewards_value = $customer->get_info($customer_id)->points;
        $customer->update_reward_points_value($customer_id, $cur_rewards_value - $payment['payment_amount']);
        $total_amount_used = floatval($total_amount_used) + floatval($payment['payment_amount']);
    }

    // L591-L601：写入 sales_payments 表

    // L603：★ 所有支付方式统一累加 total_amount
    // 含：现金、积分、礼品卡、银行卡、支票……全部计入
    $total_amount = floatval($total_amount) + floatval($payment['payment_amount']) - floatval($payment['cash_refund']);
}

// L606：用累加后的 total_amount 计算积分
$this->save_customer_rewards($customer_id, $sale_id, $total_amount, $total_amount_used);
```

**核心问题**：礼品卡支付的 `payment_amount` 被全额计入了 `$total_amount`，进而在 `save_customer_rewards` 中参与了积分累计计算。

### 12.2 三种支付方式的处理对比

| 维度 | 现金/刷卡支付 | 积分支付（Rewards） | 礼品卡支付（Giftcard） |
|------|------------|-------------------|---------------------|
| 支付时扣减余额 | 不适用 | ✅ 扣减 `customers.points` | ✅ 扣减 `giftcards.value` |
| 累加到 `total_amount_used` | ❌ | ✅ 累加 | ❌ 不累加 |
| 累加到 `total_amount` | ✅ 累加 | ✅ 累加 | ✅ **累加** |
| 参与 earned 积分计算 | ✅ | ✅ | ✅ **参与** |

### 12.3 礼品卡积分返点的合理性分析

礼品卡的积分返点是否合理，**取决于礼品卡的发放来源**：

#### 场景 A：客户用现金购买门店发售的礼品卡

- **购卡时**：客户支付 100 元现金 → `Giftcards::save_value()` 写入礼品卡记录 → **此环节不产生积分**
- **消费时**：用礼品卡消费 100 元 → 当前逻辑按 100 元累计 5 积分 → ✅ **合理**（等价于延迟的现金支付）
- **问题**：购卡时无积分，消费时才有积分——积分发放节点后移，对客户无损失

#### 场景 B：营销活动赠送礼品卡（如满 500 送 100 礼品卡）

- **赠送时**：无现金流入，直接创建礼品卡 → **此环节不产生积分**
- **消费时**：用赠送的 100 元礼品卡消费 → 当前逻辑仍按 100 元返 5 积分 → ⚠️ **不合理**（相当于无成本双重获利）
- **双重损失**：商家既付出了礼品卡成本，又额外付出了积分成本

#### 场景 C：第三方储值卡 / 企业福利卡

- **充值时**：由第三方/企业批量充值，门店无法追踪资金来源
- **消费时**：门店按实际刷卡金额与第三方结算 → 是否返积分取决于商务合同
- **当前行为**：一律按金额返积分 → 可能不符合商务约定

### 12.4 业务推演：现金+礼品卡混合支付

**示例**：订单金额 200 元，客户用礼品卡支付 80 元 + 现金支付 120 元，积分比例 5%

```
当前算法：
  total_amount = 80（礼品卡） + 120（现金） = 200
  earned = 200 × 5% = 10 积分  ★ 礼品卡 80 元也产生了 4 积分

严格返点（仅实付现金返积分）：
  total_amount = 120（仅现金部分）
  earned = 120 × 5% = 6 积分

差异：
  - 积分差额：4 积分
  - 差异率：40%（礼品卡占比越高，差异越大）
```

### 12.5 与积分支付的对称性分析

积分支付（Rewards）同样被计入 `total_amount`（见第三章 3.5 节），设计意图是"积分抵扣视为支付手段，按订单全额返点"。但礼品卡与积分有本质区别：

| 维度 | 积分（Rewards） | 礼品卡（Giftcard） |
|------|---------------|------------------|
| 是否由消费行为产生 | ✅ 每笔消费累计而来 | ❌ 发售/赠送产生 |
| 是否有对应现金流入 | ✅ 有（积分产生时的消费已支付） | ⚠️ 现金购卡时有，赠送时无 |
| 系统内可追踪来源 | ✅ 每条积分流水可追溯 | ⚠️ 仅追踪余额变动 |
| 清零规则 | 通常有有效期 | 通常无有效期 |

**结论**：积分支付计入返点基数是合理的（积分是消费的果实），但礼品卡是否计入应区分来源。

### 12.6 修复建议

**方案 A：引入配置开关（推荐，向后兼容）**

在系统配置中增加 `giftcard_earn_rewards` 开关：
- `true`（默认值，保持当前行为）：礼品卡支付金额参与积分累计
- `false`：礼品卡支付金额不计入积分累计

代码修改位置：[Sale.php L603](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Sale.php#L603)

```php
$exclude_giftcard = !$config['giftcard_earn_rewards'] 
    && !empty(strstr($payment['payment_type'], lang('Sales.giftcard')));

if (!$exclude_giftcard) {
    $total_amount = floatval($total_amount) + floatval($payment['payment_amount']) - floatval($payment['cash_refund']);
}
```

**方案 B：按礼品卡发放来源区分**

在 `giftcards` 表增加 `source_type` 字段（枚举：`cash_purchase` / `promotion` / `third_party`），消费时根据来源判断是否返积分。

优点：精确区分；缺点：需新增字段和改造购卡/发卡流程。

**方案 C：在购卡环节直接发放积分**

如果选择"礼品卡支付不返积分"，则需同步在礼品卡发售（客户现金购卡）环节增加积分累计：

```php
// Giftcards::save_value() 中新增
if ($sale_with_customer && $customer_id && $config['customer_reward_enable']) {
    // 购卡时按购卡金额发放积分
    $points_percent = $customer_rewards->get_points_percent($package_id);
    $earned = $value * $points_percent / 100;
    $customer->update_reward_points_value($customer_id, $current_points + $earned);
}
```

否则客户在"现金购卡 → 礼品卡消费"两个环节都拿不到积分，造成积分损失。

### 12.7 相关的事务回滚问题

礼品卡的 `update_giftcard_value()` 方法也是 `void` 返回值：

[Giftcard.php L307-L312](file:///d:/fz/0601-1/solo-dogfeeding/code/14-opensourcepos/app/Models/Giftcard.php#L307-L312)

```php
public function update_giftcard_value(string $giftcard_number, float $value): void
{
    $builder = $this->db->table('giftcards');
    $builder->where('giftcard_number', $giftcard_number);
    $builder->update(['value' => $value]);
}
```

与积分的 `update_reward_points_value()` 相同，SQL 执行失败时 CI4 内部 `$_trans_status` 机制保证事务回滚，但存在 `affected_rows=0`（礼品卡号不存在）的逻辑失败风险。
