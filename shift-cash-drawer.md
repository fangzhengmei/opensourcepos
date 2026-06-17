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
| [app/Models/Cashup.php](file:///d:/fz/0601-2/solo-dogfeeding/code/25-opensourcepos/app/Models/Cashup.php) | 班次模型，CRUD操作 |
| [app/Controllers/Cashups.php](file:///d:/fz/0601-2/solo-dogfeeding/code/25-opensourcepos/app/Controllers/Cashups.php) | 班次控制器，业务逻辑 |
| [app/Models/Reports/Summary_payments.php](file:///d:/fz/0601-2/solo-dogfeeding/code/25-opensourcepos/app/Models/Reports/Summary_payments.php) | 支付汇总报表，关班时对账数据源 |
| [app/Models/Expense.php](file:///d:/fz/0601-2/solo-dogfeeding/code/25-opensourcepos/app/Models/Expense.php) | 支出模型，关班时扣减现金 |
| [app/Views/cashups/form.php](file:///d:/fz/0601-2/solo-dogfeeding/code/25-opensourcepos/app/Views/cashups/form.php) | 班次表单视图（开/关班） |
| [app/Helpers/tabular_helper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/25-opensourcepos/app/Helpers/tabular_helper.php#L894-L922) | 数据行格式化辅助函数 |

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
[manage.php:53] 点击新建按钮 → 调用 getView(NEW_ENTRY)
   │
   ▼
[Cashups.php:78-103] getView($cashup_id = NEW_ENTRY)
   │
   ├─ 初始化：cashup_id == NEW_ENTRY 判定为开班
   │
   ├─ 设置默认值：
   │   ├─ open_date = 当前时间 date('Y-m-d H:i:s')
   │   ├─ close_date = open_date （临时值，关班时更新）
   │   ├─ open_employee_id = 当前登录员工ID
   │   └─ close_employee_id = 当前登录员工ID
   │
   └─ 加载 [form.php] 视图，用户输入：
       ├─ open_amount_cash（开班现金）
       ├─ transfer_amount_cash（转入/转出现金，可选）
       └─ description（备注）
   │
   ▼
用户提交表单 → 调用 postSave(NEW_ENTRY)
   │
   ▼
[Cashups.php:206-241] postSave()
   │
   ├─ 解析日期格式（根据配置的 dateformat + timeformat）
   ├─ 组装 $cash_up_data 数组
   └─ 调用 Cashup::save_value() 插入数据库
   │
   ▼
[Cashup.php:228-245] save_value()
   └─ 执行 INSERT，返回新 cashup_id
```

### 3.2 关键代码片段

**开班判定逻辑** ([Cashups.php:98-103](file:///d:/fz/0601-2/solo-dogfeeding/code/25-opensourcepos/app/Controllers/Cashups.php#L98-L103))：
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
[Cashups.php:105-183] 关班自动计算逻辑
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
   │   ├─ [Cashups.php:147] $reports_data = Summary_payments::getData($inputs)
   │   │   │
   │   │   └─ [Summary_payments.php:29-129] getData()
   │   │       ├─ 创建临时表：sumpay_taxes_temp / sumpay_items_temp / sumpay_payments_temp
   │   │       ├─ 查询 sales + sales_payments，按 payment_type 分组
   │   │       └─ 返回各支付类型的 trans_amount
   │   │
   │   └─ [Cashups.php:149-164] 遍历累加：
   │       ├─ trans_type == 'Cash' → closed_amount_cash += trans_amount
   │       ├─ trans_type == 'Due'  → closed_amount_due += trans_amount
   │       ├─ trans_type == 'Debit'/'Credit' → closed_amount_card += trans_amount
   │       └─ trans_type == 'Check' → closed_amount_check += trans_amount
   │
   ├─ 步骤5：扣减现金支出
   │   │
   │   ├─ [Cashups.php:167-180] 筛选 only_cash = true
   │   ├─ 调用 Expense::get_payments_summary()
   │   └─ 遍历：closed_amount_cash -= expense_amount
   │
   └─ 步骤6：计算总金额
       └─ [Cashups.php:182] closed_amount_total = _calculate_total(...)
   │
   ▼
加载 [form.php] 视图，显示自动计算的金额供用户确认
   │
   ▼
用户核对后提交 → postSave($cashup_id) 更新数据库
```

### 4.2 关键代码片段

**关班判定条件** ([Cashups.php:105-110](file:///d:/fz/0601-2/solo-dogfeeding/code/25-opensourcepos/app/Controllers/Cashups.php#L105-L110))：
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

**支付类型匹配** ([Cashups.php:149-164](file:///d:/fz/0601-2/solo-dogfeeding/code/25-opensourcepos/app/Controllers/Cashups.php#L149-L164))：
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

**核心方法**：[Cashups.php:280-283](file:///d:/fz/0601-2/solo-dogfeeding/code/25-opensourcepos/app/Controllers/Cashups.php#L280-L283)

```php
private function _calculate_total(
    float $open_amount_cash,
    float $transfer_amount_cash,
    float $closed_amount_due,
    float $closed_amount_cash,
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

在 [form.php:295-309](file:///d:/fz/0601-2/solo-dogfeeding/code/25-opensourcepos/app/Views/cashups/form.php#L295-L309) 中，用户修改任意金额字段时，会通过 AJAX 调用 `postAjax_cashup_total()` 实时计算并更新 `closed_amount_total`：

```javascript
$('#open_amount_cash, #transfer_amount_cash, #closed_amount_cash, #closed_amount_due, #closed_amount_card, #closed_amount_check').keyup(function() {
    $.post("cashups/ajax_cashup_total", { ... }, function(response) {
        $('#closed_amount_total').val(response.total);
    });
});
```

---

## 六、跨班交接落账处理

### 6.1 核心字段：`transfer_amount_cash`

**业务含义**（来自多语言文件）：
- 英文：`In/Out Cash`（现金进/出）
- 繁体中文：`進/出現金`
- 土耳其文：`Nakit Giriş / Çıkış`

**作用**：记录班次之间的现金转移，实现跨班交接。

### 6.2 交接场景与落账逻辑

| 场景 | transfer_amount_cash 值 | 对 closed_amount_cash 的影响 |
|------|-------------------------|------------------------------|
| 从上个班次接收现金 | 正值（+） | closed_amount_cash 增加 |
| 向下个班次移交现金 | 负值（-） | closed_amount_cash 减少 |
| 银行存款/取现 | 正/负 | 相应增减 |

### 6.3 交接落账公式

**关班时现金基数** ([Cashups.php:115](file:///d:/fz/0601-2/solo-dogfeeding/code/25-opensourcepos/app/Controllers/Cashups.php#L115))：
```php
$cash_ups_info->closed_amount_cash = $cash_ups_info->open_amount_cash + $cash_ups_info->transfer_amount_cash;
```

**完整现金计算链**：
```
关班现金 = 开班现金 + 转移金额 + 本期现金销售收入 - 本期现金支出
```

### 6.4 跨班交接流程示例

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

## 七、关班对账数据来源详解

### 7.1 Summary_payments 支付汇总

**SQL查询逻辑** ([Summary_payments.php:85-101](file:///d:/fz/0601-2/solo-dogfeeding/code/25-opensourcepos/app/Models/Reports/Summary_payments.php#L85-L101))：

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

**临时表创建** ([Summary_payments.php:135-190](file:///d:/fz/0601-2/solo-dogfeeding/code/25-opensourcepos/app/Models/Reports/Summary_payments.php#L135-L190))：
- `sumpay_taxes_temp`：税收汇总临时表
- `sumpay_items_temp`：销售商品金额临时表（含折扣计算）
- `sumpay_payments_temp`：支付金额临时表

### 7.2 Expense 现金支出汇总

**SQL查询逻辑** ([Expense.php:306-338](file:///d:/fz/0601-2/solo-dogfeeding/code/25-opensourcepos/app/Models/Expense.php#L306-L338))：

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

## 八、完整调用链路

### 8.1 开班链路

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

### 8.2 关班链路

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
    ├─ _calculate_total() 计算总金额
    └─ 加载 form.php 视图（显示自动计算结果）
        ↓
用户确认提交
    ↓
Cashups::postSave($cashup_id)
    └─ Cashup::save_value()
        └─ UPDATE cash_up
```

---

## 九、关键注意事项

1. **关班判定条件**：只有当所有 `closed_amount_*` 字段均为 0 时才会自动计算。如果用户曾经保存过半完成的关班数据，后续进入时不会重新计算。

2. **时间范围配置**：`date_or_time_format` 配置决定了对账时使用精确到日期还是精确到时间。

3. **支付类型匹配**：支付类型通过语言包的 `lang()` 函数匹配，多语言环境下需要确保配置正确。

4. **无强校验**：系统自动计算金额后，用户仍可手动修改所有 `closed_amount_*` 字段，系统不会强制校验计算结果与实际一致。

5. **软删除**：班次记录通过 `deleted` 字段软删除，而非物理删除。
