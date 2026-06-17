# 班次开闭与现金抽屉对账代码走向分析

## 一、核心概念与数据表

### 1.1 数据表结构：`ospos_cash_up`

该表存储班次（Cashup）的完整信息，包括开班、关班、各项金额和交接记录。

| 字段名 | 类型 | 说明 |
|--------|------|------|
| `cashup_id` | int(10) | 主键，自动递增 |
| `open_date` | timestamp | 开班时间 |
| `close_date` | timestamp | 关班时间 |
| `open_amount_cash` | decimal(15,2) | 开班现金金额 |
| `transfer_amount_cash` | decimal(15,2) | 现金转移金额（跨班交接） |
| `closed_amount_cash` | decimal(15,2) | 关班时现金金额 |
| `closed_amount_due` | decimal(15,2) | 关班时应收账款（赊账） |
| `closed_amount_card` | decimal(15,2) | 关班时刷卡金额 |
| `closed_amount_check` | decimal(15,2) | 关班时支票金额 |
| `closed_amount_total` | decimal(15,2) | 关班总金额 |
| `note` | int(1) | 备注标记 |
| `description` | varchar(255) | 描述 |
| `open_employee_id` | int(10) | 开班员工ID |
| `close_employee_id` | int(10) | 关班员工ID |
| `deleted` | int(1) | 软删除标记 |

### 1.2 核心代码文件

| 文件 | 说明 |
|------|------|
| `app/Models/Cashup.php` | 班次模型，CRUD操作 |
| `app/Controllers/Cashups.php` | 班次控制器，业务逻辑 |
| `app/Models/Reports/Summary_payments.php` | 支付汇总报表，关班时对账数据源 |
| `app/Models/Expense.php` | 支出模型，关班时扣减现金 |
| `app/Views/cashups/form.php` | 班次表单视图（开/关班） |
| `app/Helpers/tabular_helper.php` | 数据行格式化辅助函数 |

---

## 二、代码调用总流程图

```
用户操作
   │
   ▼
┌─────────────────────────────────────────────────────────────┐
│                   路由：Cashups 控制器                       │
└─────────────────────────────────────────────────────────────┘
   │
   ├─ 开班：getIndex() → getView(NEW_ENTRY) → 表单 → postSave()
   │
   └─ 关班：getView($cashup_id) → 自动计算 → 表单 → postSave()
         │
         ├─ Summary_payments::getData()  ── 销售支付汇总
         │   └─ sales_payments 表按支付类型分组统计
         │
         └─ Expense::get_payments_summary()  ── 现金支出汇总
             └─ expenses 表筛选现金支付的支出
```

---

## 三、开班流程（Open Shift）

### 3.1 代码走向

```
用户点击「New Cashup」
   │
   ▼
manage.php 点击新建按钮 → 调用 getView(NEW_ENTRY)
   │
   ▼
Cashups.php: getView($cashup_id = NEW_ENTRY)
   │
   ├─ 初始化：cashup_id == NEW_ENTRY 判定为开班
   │
   ├─ 设置默认值：
   │   ├─ open_date = 当前时间 date('Y-m-d H:i:s')
   │   ├─ close_date = open_date （临时值，关班时更新）
   │   ├─ open_employee_id = 当前登录员工ID
   │   └─ close_employee_id = 当前登录员工ID
   │
   └─ 加载 form.php 视图，用户输入：
       ├─ open_amount_cash（开班现金）
       ├─ transfer_amount_cash（转入/转出现金，可选）
       └─ description（备注）
   │
   ▼
用户提交表单 → 调用 postSave(NEW_ENTRY)
   │
   ▼
Cashups.php: postSave()
   │
   ├─ 解析日期格式（根据配置的 dateformat + timeformat）
   ├─ 组装 $cash_up_data 数组
   └─ 调用 Cashup::save_value() 插入数据库
   │
   ▼
Cashup.php: save_value()
   └─ 执行 INSERT，返回新 cashup_id
```

### 3.2 关键代码片段

**开班判定逻辑**（`app/Controllers/Cashups.php`，第98-103行）：
```php
if ($cash_ups_info->cashup_id == NEW_ENTRY) {
    $cash_ups_info->open_date = date('Y-m-d H:i:s');
    $cash_ups_info->close_date = $cash_ups_info->open_date;
    $cash_ups_info->open_employee_id = $this->employee->get_logged_in_employee_info()->person_id;
    $cash_ups_info->close_employee_id = $this->employee->get_logged_in_employee_info()->person_id;
}
```

---

## 四、关班流程（Close Shift）与余额校验

### 4.1 代码走向

```
用户点击班次列表的「Edit」按钮
   │
   ▼
调用 getView($cashup_id) （$cashup_id != NEW_ENTRY）
   │
   ▼
Cashups.php 第105-183行：关班自动计算逻辑
   │
   ├─ 判定条件：所有 closed_amount 字段均为 0 或 null
   │   (closed_amount_cash/due/card/check == 0)
   │
   ├─ 步骤1：设置关班时间
   │   └─ close_date = date('Y-m-d H:i:s')
   │
   ├─ 步骤2：初始化现金基数
   │   └─ closed_amount_cash = open_amount_cash + transfer_amount_cash
   │
   ├─ 步骤3：根据 date_or_time_format 配置，
   │   │      构造查询时间范围 $inputs
   │   │
   │   └─ 日期模式：仅用日期部分（YYYY-MM-DD）
   │   └─ 日期时间模式：用完整时间戳
   │
   ├─ 步骤4：获取销售支付汇总
   │   │
   │   ├─ $reports_data = Summary_payments::getData($inputs)
   │   │   │
   │   │   └─ Summary_payments.php: getData()
   │   │       ├─ 创建临时表：sumpay_taxes_temp / sumpay_items_temp / sumpay_payments_temp
   │   │       ├─ 查询 sales + sales_payments，按 payment_type 分组
   │   │       └─ 返回各支付类型的 trans_amount
   │   │
   │   └─ 遍历累加：
   │       ├─ trans_type == 'Cash' → closed_amount_cash += trans_amount
   │       ├─ trans_type == 'Due'  → closed_amount_due += trans_amount
   │       ├─ trans_type == 'Debit'/'Credit' → closed_amount_card += trans_amount
   │       └─ trans_type == 'Check' → closed_amount_check += trans_amount
   │
   ├─ 步骤5：扣减现金支出
   │   │
   │   ├─ 筛选 only_cash = true
   │   ├─ 调用 Expense::get_payments_summary()
   │   └─ 遍历：closed_amount_cash -= expense_amount
   │
   └─ 步骤6：计算总金额
       └─ closed_amount_total = _calculate_total(...)
   │
   ▼
加载 form.php 视图，显示自动计算的金额供用户确认
   │
   ▼
用户核对后提交 → postSave($cashup_id) 更新数据库
```

### 4.2 关键代码片段

**关班判定条件**（`app/Controllers/Cashups.php`，第105-110行）：
```php
elseif (
    floatval($cash_ups_info->closed_amount_cash) == 0
    && floatval($cash_ups_info->closed_amount_due) == 0
    && floatval($cash_ups_info->closed_amount_card) == 0
    && floatval($cash_ups_info->closed_amount_check) == 0
) {
    // 进入关班自动计算流程
}
```

**支付类型匹配**（`app/Controllers/Cashups.php`，第149-164行）：
```php
foreach ($reports_data as $row) {
    if ($row['trans_group'] == lang('Reports.trans_payments')) {
        if ($row['trans_type'] == lang('Sales.cash')) {
            $cash_ups_info->closed_amount_cash += $row['trans_amount'];
        } elseif ($row['trans_type'] == lang('Sales.due')) {
            $cash_ups_info->closed_amount_due += $row['trans_amount'];
        } elseif ($row['trans_type'] == lang('Sales.debit') || $row['trans_type'] == lang('Sales.credit')) {
            $cash_ups_info->closed_amount_card += $row['trans_amount'];
        } elseif ($row['trans_type'] == lang('Sales.check')) {
            $cash_ups_info->closed_amount_check += $row['trans_amount'];
        }
    }
}
```

---

## 五、总金额计算公式

### 5.1 公式定义

**核心方法**：`_calculate_total()`（位于 `app/Controllers/Cashups.php`，第280-283行）

```php
private function _calculate_total(
    float $open_amount_cash,
    float $transfer_amount_cash,
    float $closed_amount_due,      // 注意：第3个参数是 due
    float $closed_amount_cash,     // 注意：第4个参数是 cash
    float $closed_amount_card,
    $closed_amount_check
): float {
    return ($closed_amount_cash - $open_amount_cash - $transfer_amount_cash 
            + $closed_amount_due + $closed_amount_card + $closed_amount_check);
}
```

### 5.2 公式解读

```
closed_amount_total = 
    (closed_amount_cash - open_amount_cash - transfer_amount_cash)  // 现金净增减
    + closed_amount_due     // 赊账金额
    + closed_amount_card    // 刷卡金额
    + closed_amount_check   // 支票金额
```

**含义**：
- `closed_amount_cash - open_amount_cash - transfer_amount_cash`：本期现金的实际增减（剔除期初和转移）
- 加上其他支付方式的金额，得到本期的总营业收入

### 5.3 前端实时计算

在 `app/Views/cashups/form.php` 中，用户修改任意金额字段时，会通过 AJAX 调用 `postAjax_cashup_total()` 实时计算并更新 `closed_amount_total`：

```javascript
$('#open_amount_cash, #transfer_amount_cash, #closed_amount_cash, #closed_amount_due, #closed_amount_card, #closed_amount_check').keyup(function() {
    $.post("cashups/ajax_cashup_total", { ... }, function(response) {
        $('#closed_amount_total').val(response.total);
    });
});
```

---

## 六、现金与应收金额的传递顺序不一致问题

### 6.1 问题发现

`_calculate_total()` 函数在代码中有两处调用，但两处的参数传递顺序不一致，**现金（cash）和应收（due）的位置被颠倒**。

### 6.2 两处调用的对比

**调用点1：关班自动计算（`getView()` 方法，第182行）**

```php
// 参数顺序：open, transfer, CASH, DUE, card, check
$cash_ups_info->closed_amount_total = $this->_calculate_total(
    $cash_ups_info->open_amount_cash,       // 参数1
    $cash_ups_info->transfer_amount_cash,   // 参数2
    $cash_ups_info->closed_amount_cash,     // 参数3 ← cash（第3位）
    $cash_ups_info->closed_amount_due,      // 参数4 ← due（第4位）
    $cash_ups_info->closed_amount_card,     // 参数5
    $cash_ups_info->closed_amount_check     // 参数6
);
```

**调用点2：前端AJAX实时计算（`postAjax_cashup_total()` 方法，第272行）**

```php
// 参数顺序：open, transfer, DUE, CASH, card, check
$total = $this->_calculate_total(
    $open_amount_cash,       // 参数1
    $transfer_amount_cash,   // 参数2
    $closed_amount_due,      // 参数3 ← due（第3位）
    $closed_amount_cash,     // 参数4 ← cash（第4位）
    $closed_amount_card,     // 参数5
    $closed_amount_check     // 参数6
);
```

**函数定义的参数顺序**（第280行）：

```php
private function _calculate_total(
    float $open_amount_cash,     // 参数1
    float $transfer_amount_cash, // 参数2
    float $closed_amount_due,    // 参数3 ← due（第3位）
    float $closed_amount_cash,   // 参数4 ← cash（第4位）
    float $closed_amount_card,   // 参数5
    $closed_amount_check         // 参数6
): float {
```

### 6.3 不一致汇总表

| 参数位置 | 函数定义 | 调用点1（getView） | 调用点2（AJAX） |
|---------|---------|-------------------|-----------------|
| 参数1 | open_amount_cash | open_amount_cash ✓ | open_amount_cash ✓ |
| 参数2 | transfer_amount_cash | transfer_amount_cash ✓ | transfer_amount_cash ✓ |
| 参数3 | closed_amount_due | closed_amount_cash ✗（错位） | closed_amount_due ✓ |
| 参数4 | closed_amount_cash | closed_amount_due ✗（错位） | closed_amount_cash ✓ |
| 参数5 | closed_amount_card | closed_amount_card ✓ | closed_amount_card ✓ |
| 参数6 | closed_amount_check | closed_amount_check ✓ | closed_amount_check ✓ |

### 6.4 对账目核对的影响

#### 6.4.1 数值结果的等价性

由于加法交换律，虽然参数顺序颠倒了，但最终 `closed_amount_total` 的**数值结果在数学上是相同的**：

```
// 正确顺序（调用点2 / 函数定义）
total = cash - open - transfer + due + card + check

// 颠倒顺序（调用点1）
// 函数内的 $closed_amount_due 实际是 cash 值
// 函数内的 $closed_amount_cash 实际是 due 值
total = due - open - transfer + cash + card + check

// 两者相等
cash + due = due + cash  （加法交换律）
```

#### 6.4.2 对账目核对的实际影响

尽管数值结果相同，但这种不一致给账目核对和系统维护带来了多方面的问题：

1. **语义混淆与理解成本**
   - 阅读和维护代码时，容易对 cash 和 due 的含义产生误解
   - 新人接手时容易搞错参数含义，引入新的 bug
   - 参数顺序与业务直觉（cash 在前 due 在后 / due 在前 cash 在后）不一致

2. **调试与排错困难**
   - 当 total 金额出现异常时，排查者可能会因参数错位而误判问题根源
   - 在断点调试时，变量名与实际值不匹配，增加调试难度

3. **未来扩展风险**
   - 如果未来公式需要区分 cash 和 due 的权重（例如 cash 乘以某个系数），这种错位会直接导致计算错误
   - 如果新增与 cash/due 相关的校验逻辑，错位会导致校验失效

4. **两处入口结果一致性的隐性依赖**
   - 当前两处入口结果一致纯属"巧合"（依赖加法交换律）
   - 一旦公式逻辑稍有变动，两处入口可能产生不一致的结果
   - 这意味着用户在页面上看到的自动计算值（来自 getView）和手动修改后 AJAX 计算的值，虽然目前相同，但本质上走了不同的"参数映射路径"

5. **对账逻辑的可信度下降**
   - 核心对账公式存在参数错位，降低了对账系统的整体可信度
   - 审计或财务核对时，这种代码质量问题可能引发对系统准确性的怀疑

### 6.5 代码中的迹象

代码中已经留下了开发者意识到此问题的痕迹：

- `_calculate_total` 函数上方的 TODO 注释：`need to get rid of hungarian notation here. Also, the signature is pretty long. Perhaps they need to go into an object or array?`
- `postAjax_cashup_total` 方法中的 TODO 注释：`hungarian notation`

这表明开发者已意识到参数过多和命名问题，但尚未修复参数顺序不一致的隐患。

---

## 七、跨班交接落账处理

### 7.1 核心字段：`transfer_amount_cash`

**业务含义**（来自多语言文件）：
- 英文：`In/Out Cash`（现金进/出）
- 繁体中文：`進/出現金`
- 土耳其文：`Nakit Giriş / Çıkış`

**作用**：记录班次之间的现金转移，实现跨班交接。

### 7.2 交接场景与落账逻辑

| 场景 | transfer_amount_cash 值 | 对 closed_amount_cash 的影响 |
|------|-------------------------|------------------------------|
| 从上个班次接收现金 | 正值（+） | closed_amount_cash 增加 |
| 向下个班次移交现金 | 负值（-） | closed_amount_cash 减少 |
| 银行存款/取现 | 正/负 | 相应增减 |

### 7.3 交接落账公式

**关班时现金基数**（`app/Controllers/Cashups.php`，第115行）：
```php
$cash_ups_info->closed_amount_cash = $cash_ups_info->open_amount_cash + $cash_ups_info->transfer_amount_cash;
```

**完整现金计算链**：
```
关班现金 = 开班现金 + 转移金额 + 本期现金销售收入 - 本期现金支出
```

### 7.4 跨班交接流程示例

```
班次A（早班）                          班次B（晚班）
   │                                   │
   ├─ open_amount_cash = 500           ├─ open_amount_cash = 100
   ├─ transfer_amount_cash = -400      ├─ transfer_amount_cash = +400
   │  (移交400给晚班)                   │  (接收早班400)
   │                                   │
   └─ closed_amount_cash               └─ closed_amount_cash
      = 500 + (-400) + 销售 - 支出        = 100 + 400 + 销售 - 支出
      = 100 + 销售 - 支出                 = 500 + 销售 - 支出
```

---

## 八、关班对账数据来源详解

### 8.1 Summary_payments 支付汇总

**SQL查询逻辑**（`app/Models/Reports/Summary_payments.php`，第85-101行）：

```sql
SELECT 
    'trans_payments' AS trans_group,
    sales_payments.payment_type as trans_type,
    COUNT(sales.sale_id) AS trans_sales,
    SUM(payment_amount - cash_refund) AS trans_amount
FROM sales
LEFT JOIN sales_payments ON sales.sale_id = sales_payments.sale_id
WHERE 
    sales.sale_status = COMPLETED
    AND sale_time BETWEEN {open_date} AND {close_date}
GROUP BY sales_payments.payment_type
```

**临时表创建**（`app/Models/Reports/Summary_payments.php`，第135-190行）：
- `sumpay_taxes_temp`：税收汇总临时表
- `sumpay_items_temp`：销售商品金额临时表（含折扣计算）
- `sumpay_payments_temp`：支付金额临时表

### 8.2 Expense 现金支出汇总

**SQL查询逻辑**（`app/Models/Expense.php`，第306-338行）：

```sql
SELECT 
    payment_type, 
    COUNT(amount) AS count, 
    SUM(amount) AS amount
FROM expenses
WHERE 
    deleted = 0
    AND date BETWEEN {open_date} AND {close_date}
    AND payment_type LIKE '%Cash%'
GROUP BY payment_type
```

---

## 九、完整调用链路

### 9.1 开班链路

```
Cashups::getIndex()
    ↓
Cashups::getView(NEW_ENTRY)
    ├─ 设置 open_date / open_employee_id
    └─ 加载 form.php 视图
        ↓
用户提交表单
    ↓
Cashups::postSave(NEW_ENTRY)
    ├─ 解析日期
    ├─ parse_decimals() 处理金额
    └─ Cashup::save_value()
        └─ INSERT INTO cash_up
```

### 9.2 关班链路

```
Cashups::getView($cashup_id)
    ├─ 判定为关班（closed_amount 均为 0）
    ├─ 设置 close_date / close_employee_id
    ├─ 初始化 closed_amount_cash = open + transfer
    ├─ Summary_payments::getData()
    │   └─ create_summary_payments_temp_tables()
    │       ├─ 临时表1：税收
    │       ├─ 临时表2：商品金额
    │       └─ 临时表3：支付
    ├─ 按支付类型累加 closed_amount_*
    ├─ Expense::get_payments_summary(only_cash=true)
    ├─ 扣减现金支出
    ├─ _calculate_total() 计算总金额  ← 注意：此处参数顺序与函数定义不一致
    └─ 加载 form.php 视图（显示自动计算结果）
        ↓
用户修改金额 → AJAX 调用 postAjax_cashup_total()  ← 此处参数顺序与函数定义一致
        ↓
用户确认提交
    ↓
Cashups::postSave($cashup_id)
    └─ Cashup::save_value()
        └─ UPDATE cash_up
```

---

## 十、关键注意事项

1. **关班判定条件**：只有当所有 `closed_amount_*` 字段均为 0 时才会自动计算。如果用户曾经保存过半完成的关班数据，后续进入时不会重新计算。

2. **时间范围配置**：`date_or_time_format` 配置决定了对账时使用精确到日期还是精确到时间。

3. **支付类型匹配**：支付类型通过语言包的 `lang()` 函数匹配，多语言环境下需要确保配置正确。

4. **无强校验**：系统自动计算金额后，用户仍可手动修改所有 `closed_amount_*` 字段，系统不会强制校验计算结果与实际一致。

5. **软删除**：班次记录通过 `deleted` 字段软删除，而非物理删除。

6. **参数顺序不一致（重要）**：`_calculate_total()` 函数在 `getView()` 和 `postAjax_cashup_total()` 两处调用中，cash 和 due 参数的传递顺序相反。当前因加法交换律数值结果一致，但存在维护风险，详见第六章。
