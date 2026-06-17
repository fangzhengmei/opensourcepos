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

## 七、关班金额重算边界与手工修正影响

### 7.1 两种计算机制

班次金额计算存在两套独立的机制，触发条件和计算范围完全不同：

| 机制 | 触发方式 | 计算范围 | 数据来源 |
|------|---------|---------|---------|
| **服务端自动汇总** | 进入关班页面时，满足条件则自动触发 | 6个金额字段全部重算 | Summary_payments + Expense 真实业务数据 |
| **前端AJAX实时计算** | 用户修改表单输入框时触发（keyup） | 仅重算 closed_amount_total | 表单上的6个金额字段（纯数学公式） |

### 7.2 服务端自动汇总的触发边界

**入口**：`Cashups::getView($cashup_id)`（第104-183行）

#### 7.2.1 触发条件（全部满足）

```php
if (
    floatval($cash_ups_info->closed_amount_cash) == 0
    && floatval($cash_ups_info->closed_amount_due) == 0
    && floatval($cash_ups_info->closed_amount_card) == 0
    && floatval($cash_ups_info->closed_amount_check) == 0
) {
    // 执行自动汇总
}
```

**四个字段必须全部为 0**（包括 null，因为 `floatval(null) == 0`）。

#### 7.2.2 自动汇总的完整计算流程

```
Step 1: 设置 close_date = 当前时间
Step 2: closed_amount_cash = open_amount_cash + transfer_amount_cash
        （注意：cash初始化为 open+transfer，其他 due/card/check 初始为0）
Step 3: 构造时间范围 inputs（受 date_or_time_format 配置影响）
Step 4: Summary_payments::getData() 按支付类型汇总
        → 分别累加至 closed_amount_cash/due/card/check
Step 5: Expense::get_payments_summary(only_cash=true) 汇总现金支出
        → 从 closed_amount_cash 中扣除
Step 6: _calculate_total() 计算 closed_amount_total
```

#### 7.2.3 不触发自动汇总的情形

只要四个 closed_amount 字段中**任意一个不为 0**，就**完全跳过**自动汇总：

- 已经保存过关班数据的班次 → 不再重算
- 用户手工修改过其中任何一个金额并保存 → 不再重算
- 数据库中因历史数据导致某字段有非零值 → 不再重算

**重要结论**：自动汇总只执行「从零构建」一次，之后就是纯手工维护模式。

### 7.3 前端AJAX实时计算的边界

**入口**：`form.php` 第295-309行 → `Cashups::postAjax_cashup_total()`（第263-275行）

#### 7.3.1 触发条件

6个输入框中任何一个的 `keyup` 事件：
- `open_amount_cash`
- `transfer_amount_cash`
- `closed_amount_cash`
- `closed_amount_due`
- `closed_amount_card`
- `closed_amount_check`

#### 7.3.2 计算范围

**只重算 closed_amount_total**，其他5个输入框的值保持不变。

```
用户修改任意金额字段 → AJAX提交6个金额 
    → 服务端 _calculate_total() 纯公式计算 
        → 返回 total 字符串
            → 前端更新 closed_amount_total 显示值
```

#### 7.3.3 关键特征

1. **纯数学公式**：不查询销售、支出等业务数据，仅基于表单上的6个数值
2. **仅影响显示**：AJAX计算结果只更新页面显示的 `closed_amount_total`，不自动保存到数据库
3. **保存时原样写入**：`postSave()` 直接读取 `$_POST['closed_amount_total']` 保存，不做校验
4. **参数顺序正确**：AJAX 调用中 cash 和 due 参数顺序与函数定义一致（不同于 getView 中的错位问题）

### 7.4 两种机制的对比

| 对比项 | 服务端自动汇总 | 前端AJAX实时计算 |
|--------|-------------|-----------------|
| 触发时机 | 首次进入关班页面 | 修改任一字段时 |
| 触发条件 | closed_amount 全为0 | 输入框 keyup 事件 |
| 数据来源 | 数据库真实销售+支出 | 表单上的6个数值 |
| 重写字段 | closed_amount_cash/due/card/check/total 全部 | 仅 closed_amount_total |
| 是否保存 | 需用户提交后才保存 | 不保存，仅显示 |
| 可重复执行 | 仅一次（保存后不再触发） | 每次 keyup 都触发 |
| 与业务数据一致性 | 强一致（基于真实数据） | 不保证（基于表单值） |

### 7.5 班后手工修正对金额的影响

"班后"指班次已保存关班数据（即 closed_amount_* 不全为0）后，再次进入编辑页面进行修改的情形。

#### 7.5.1 修改不同字段的影响矩阵

| 修改的字段 | closed_amount_cash | closed_amount_due | closed_amount_card | closed_amount_check | closed_amount_total | 触发自动重算？ |
|-----------|--------------------|-------------------|--------------------|---------------------|---------------------|---------------|
| open_amount_cash | ❌ 不变（需手动改） | ❌ 不变 | ❌ 不变 | ❌ 不变 | ✅ AJAX自动重算 | ❌ 否 |
| transfer_amount_cash | ❌ 不变（需手动改） | ❌ 不变 | ❌ 不变 | ❌ 不变 | ✅ AJAX自动重算 | ❌ 否 |
| closed_amount_cash | ✅ 用户输入值 | ❌ 不变 | ❌ 不变 | ❌ 不变 | ✅ AJAX自动重算 | ❌ 否 |
| closed_amount_due | ❌ 不变 | ✅ 用户输入值 | ❌ 不变 | ❌ 不变 | ✅ AJAX自动重算 | ❌ 否 |
| closed_amount_card | ❌ 不变 | ❌ 不变 | ✅ 用户输入值 | ❌ 不变 | ✅ AJAX自动重算 | ❌ 否 |
| closed_amount_check | ❌ 不变 | ❌ 不变 | ❌ 不变 | ✅ 用户输入值 | ✅ AJAX自动重算 | ❌ 否 |
| close_date | ❌ 不变 | ❌ 不变 | ❌ 不变 | ❌ 不变 | ❌ 不变 | ❌ 否 |

**核心结论**：班后修改 `open_amount_cash` 或 `transfer_amount_cash`，**不会**自动触发 closed_amount_cash 的重新汇总计算。用户必须手动同步修改 closed_amount_cash，否则 total 的计算基于旧的 cash 值，会产生"total 公式对但与实际业务不符"的隐性错误。

#### 7.5.2 典型场景分析

**场景1：班后发现 transfer_amount_cash 填错了**

```
初始状态：
  open_amount_cash = 500
  transfer_amount_cash = 0   ← 忘记填交接的 200
  closed_amount_cash = 1500  ← 已关班时自动计算的值（500+0+销售-支出）
  closed_amount_total = 1500 - 500 - 0 + ... = 1000

用户操作：
  进入编辑页面 → 将 transfer_amount_cash 改为 200

实际结果：
  closed_amount_cash 仍然是 1500 ← 不会自动重算
  closed_amount_total 变为 1500 - 500 - 200 + ... = 800 ← AJAX 自动改了 total
  但 cash 的实际构成（500+200+销售-支出=1500 不对，应该是 700+销售-支出）
  → 账目逻辑自相矛盾
```

**场景2：班后发现 open_amount_cash 填错了**

类似场景1，修改 open 后 total 会 AJAX 重算，但 closed_amount_cash 本身不变，导致"期末现金 - 期初现金"的差额不真实。

**场景3：班后补填一笔漏记的现金支出**

系统没有提供"添加支出自动刷新 cash"的联动。用户必须：
1. 手动去支出模块添加支出记录
2. 手动回到班次编辑页修改 closed_amount_cash
3. total 会 AJAX 自动更新，但 cash 的正确性全靠人工

### 7.6 强制触发重新自动汇总的方法

如果确实需要基于最新业务数据重新计算关班金额，需要满足"四个 closed_amount 全为0"的条件。实际操作路径：

**方法一：手工清零法（推荐）**
1. 进入班次编辑页面
2. 手动将 `closed_amount_cash`、`closed_amount_due`、`closed_amount_card`、`closed_amount_check` 四个字段全部改为 0
3. 保存
4. 再次进入该班次编辑页面 → 触发自动重算

**方法二：数据库直接更新法**
```sql
UPDATE cash_up 
SET closed_amount_cash = 0, 
    closed_amount_due = 0,
    closed_amount_card = 0,
    closed_amount_check = 0,
    closed_amount_total = 0
WHERE cashup_id = {班次ID};
```
然后在前端重新进入编辑页面。

**注意**：重新触发自动汇总后，之前手工修改过的任何 closed_amount 值都会被覆盖为系统计算值。

### 7.7 保存时的行为

**入口**：`postSave()` 第206-241行

保存逻辑非常"透明"——不做任何校验或重算：

1. 从 `$_POST` 读取所有12个字段的值
2. `parse_decimals()` 格式化金额
3. 直接调用 `Cashup::save_value()` 执行 INSERT 或 UPDATE

**关键点**：
- `closed_amount_total` 直接保存 POST 过来的值，不重新调用 `_calculate_total()` 校验
- 没有一致性校验（如 total 是否等于 cash - open - transfer + due + card + check）
- 没有业务校验（如 cash 是否等于 open + transfer + 销售 - 支出）
- 完全信任前端传来的数据

---

## 八、跨班交接落账处理

### 8.1 核心字段：`transfer_amount_cash`

**业务含义**（来自多语言文件）：
- 英文：`In/Out Cash`（现金进/出）
- 繁体中文：`進/出現金`
- 土耳其文：`Nakit Giriş / Çıkış`

**字段本质**：`transfer_amount_cash` 是班次记录上的一个**单体数值字段**，而非独立的转移记录实体。它直接存储在 `ospos_cash_up` 表中，与班次一一绑定。

### 8.2 代码对该字段的全流程处理

#### 8.2.1 保存流程（开班时）

**入口**：`Cashups::postSave(NEW_ENTRY)`（第206-241行）

```php
// 第218行：直接从表单读取，不做任何额外处理
'transfer_amount_cash' => parse_decimals($this->request->getPost('transfer_amount_cash')),
```

**处理逻辑**：
1. 从 POST 表单 `transfer_amount_cash` 字段读取原始值
2. 通过 `parse_decimals()` 将本地化的金额格式（如千分位、货币符号）转换为纯数值
3. 直接写入 `cash_up_data` 数组，通过 `Cashup::save_value()` 执行 INSERT

**关键结论**：保存时**完全不校验**该值与其他班次的关联性，不做任何正负值限制，也不自动读取上个班次的数据作为默认值。

#### 8.2.2 表单渲染流程

**入口**：`app/Views/cashups/form.php`（第65-82行）

```php
// 第72-77行：渲染 transfer_amount_cash 输入框
<?= form_input([
    'name'  => 'transfer_amount_cash',
    'id'    => 'transfer_amount_cash',
    'class' => 'form-control input-sm',
    'value' => to_currency_no_money($cash_ups_info->transfer_amount_cash)
]) ?>
```

**处理逻辑**：
1. 开班时（NEW_ENTRY）：`$cash_ups_info->transfer_amount_cash` 为 `0`（由 `Cashup::getEmptyObject()` 初始化，第215行）
2. 关班时：直接回显数据库中该班次已保存的值
3. 用户可手动输入任意正/负值

**关键结论**：前端表单中，`transfer_amount_cash` 的初始值始终为 `0`（开班）或「本班次之前保存的值」（关班），**不会自动回填**任何来自其他班次的数据。

#### 8.2.3 列表查询流程

**入口**：`Cashup::search()`（第85-159行）

```sql
-- 第106行：查询中仅做 MAX 聚合，不做跨表关联
MAX(cash_up.transfer_amount_cash) AS transfer_amount_cash
```

**列表展示**：`tabular_helper.php` 第903行
```php
'transfer_amount_cash' => to_currency($cash_up->transfer_amount_cash),
```

**关键结论**：查询列表时，`transfer_amount_cash` 仅作为本班次的单体字段展示，**不进行跨班次比对或关联查询**，不会在UI上提示"该值与其他班次是否匹配"。

#### 8.2.4 关班自动计算流程

**入口**：`Cashups::getView($cashup_id)` 关班逻辑（第115行）

```php
// 第115行：作为现金基数的一部分参与计算
$cash_ups_info->closed_amount_cash = $cash_ups_info->open_amount_cash + $cash_ups_info->transfer_amount_cash;
```

后续步骤（第147-180行）：
1. 在此基数上累加 `Summary_payments` 中的现金销售收入（Cash）
2. 扣除 `Expense` 中的现金支出

**计算链**：
```
closed_amount_cash（关班计算结果）
    = open_amount_cash + transfer_amount_cash + 本期现金销售收入 - 本期现金支出
```

**关键结论**：`transfer_amount_cash` 作为计算的"期初调整项"参与关班现金计算，**但其值的正确性完全依赖用户手工输入**，系统不验证来源或去向。

#### 8.2.5 总金额计算中的角色

**两处入口中均正确传递**：`_calculate_total()` 的参数中，`transfer_amount_cash` 始终作为第二个参数传入（不同于 cash/due 的错位问题），不存在顺序不一致。

### 8.3 代码是否自动关联前后班次？结论

**明确结论：代码完全不做任何自动关联。**

#### 8.3.1 具体证据

| 检查项 | 结论 | 证据位置 |
|--------|------|---------|
| 新建班次时自动读取上一班次的 `closed_amount_cash` 作为 `open_amount_cash` 默认值？ | ❌ 否 | `getEmptyObject()` 初始化为0，`getView(NEW_ENTRY)` 中未查询其他班次 |
| 新建班次时自动读取上一班次的转出金额作为本班次 `transfer_amount_cash` 默认值？ | ❌ 否 | `getView(NEW_ENTRY)` 中无相关逻辑 |
| 保存时校验本班次的 `transfer_amount_cash` 是否能在其他班次找到对应相反数？ | ❌ 否 | `postSave()` 仅做 `parse_decimals()`，不做跨班次校验 |
| 是否存在独立的「现金转移记录表」存储配对转移信息？ | ❌ 否 | 数据库仅有 `cash_up` 表单体字段，无 `cash_transfer` 等关联表 |
| 关班时是否生成下一班次的待匹配记录？ | ❌ 否 | 关班仅 UPDATE 本记录，不 INSERT 任何关联数据 |
| 列表查询时是否高亮显示"不匹配"的转移金额？ | ❌ 否 | `search()` 方法无跨班次 JOIN 或 HAVING 比对逻辑 |

#### 8.3.2 代码中的相关线索

在 `app/Models/Cashup.php` 的 `$allowedFields`（第20-35行）中可以看到：
```php
protected $allowedFields = [
    'open_date', 'close_date',
    'open_cash_amount',      // ← 注意：此字段名与数据库列名 open_amount_cash 不一致
    'transfer_cash_amount',  // ← 注意：此字段名与数据库列名 transfer_amount_cash 不一致
    'note', ...
];
```

`$allowedFields` 中使用了 `open_cash_amount` 和 `transfer_cash_amount`，但实际数据库列名是 `open_amount_cash` 和 `transfer_amount_cash`。虽然 `save_value()` 中使用了自定义 INSERT/UPDATE 而非 CI4 的 `save()` 方法因此不触发错误，但这也反映出该模块在字段命名上存在历史遗留的不一致，进一步说明**跨班关联逻辑未被系统化设计**。

### 8.4 交接场景与实际落账逻辑

| 场景 | 手动操作 | transfer_amount_cash 值 | 对 closed_amount_cash 的影响 |
|------|---------|-------------------------|------------------------------|
| 从上个班次接收现金 | 本班次开班时手工填写 | 正值（+） | closed_amount_cash 增加 |
| 向下个班次移交现金 | 本班次关班时手工填写 | 负值（-） | closed_amount_cash 减少 |
| 银行存款/取现 | 发生时在当班次填写 | 存（-）/ 取（+） | 相应增减 |
| 其他现金进出（如备用金） | 手工填写 | 正/负 | 相应增减 |

### 8.5 交接落账公式

**关班时现金基数初始化**（`app/Controllers/Cashups.php`，第115行）：
```php
$cash_ups_info->closed_amount_cash = $cash_ups_info->open_amount_cash + $cash_ups_info->transfer_amount_cash;
```

**完整现金计算链**：
```
关班现金（closed_amount_cash）
    = 开班现金（open_amount_cash）
    + 转移金额（transfer_amount_cash）
    + 本期现金销售收入（Summary_payments 中 Cash 类型汇总）
    - 本期现金支出（Expense 中仅 Cash 类型汇总）
```

### 8.6 跨班交接的实际操作模式（非自动化）

由于系统不做自动关联，实际业务中跨班交接必须**完全依赖手工配对操作**：

```
步骤1：班次A（早班）关班
   │
   ├─ 本班次 open_amount_cash = 500
   ├─ 如需移交 400 给晚班 → 关班时手工填写 transfer_amount_cash = -400
   ├─ 系统计算 closed_amount_cash = 500 + (-400) + 销售 - 支出
   └─ 保存关班
         │
         ▼
步骤2：班次B（晚班）开班
   │
   ├─ open_amount_cash 默认为 0，需手工填写（如留底100 → 填 100）
   ├─ 需知道早班移交了 400 → 手工填写 transfer_amount_cash = +400
   └─ 保存开班
         │
         ▼
步骤3：班次B关班
   │
   └─ 系统自动计算 closed_amount_cash = 100 + 400 + 销售 - 支出
```

**配对风险**：
- 若班次A忘记填 `-400`，但班次B填了 `+400` → 两班账目各自不平，系统无任何提示
- 若班次A填了 `-400`，但班次B填错为 `+4000` → 数值不匹配，系统不告警
- 若班次A填了 `-400`，班次B忘了填 `+400` → 不匹配，系统不校验

### 8.7 对账目核对的影响

1. **无强制一致性保障**：两个班次之间的转移金额是否相反数，完全依赖人工操作，系统不提供数据库级或应用级的一致性校验。

2. **无法生成配对审计追踪**：由于没有独立的转移记录表，无法查询"哪两个班次之间发生了转移"，也无法查询"某笔转移是否已被对方确认接收"。

3. **期末对账难度**：如果发生转移漏填、错填，只能通过逐班次比对 `transfer_amount_cash` 来人工排查，没有快捷的差异报告。

4. **`description` 字段的实际用途**：由于系统不提供关联机制，实际操作中需要在 `description` 文本字段中用自然语言备注「移交XX给班次Y」或「接收班次X的XX」来辅助人工核对。

5. **修正方式**：若发现之前班次的 transfer 填错，必须重新编辑该班次记录（`postSave($cashup_id)`）进行修正，因为后续班次不会自动联动。修正后若已关班，需要重新进入关班页面触发重新计算（但仅在 `closed_amount_*` 全为0时才会重算，已关班的需先清空金额才能触发，实际操作中通常只能手工修改 closed_amount_cash）。

---

## 九、关班对账数据来源详解

### 9.1 Summary_payments 支付汇总

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

### 9.2 Expense 现金支出汇总

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

## 十、完整调用链路

### 10.1 开班链路

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

### 10.2 关班链路

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

## 十一、关键注意事项

1. **关班判定条件**：只有当所有 `closed_amount_*` 字段均为 0 时才会自动计算。如果用户曾经保存过半完成的关班数据，后续进入时不会重新计算。

2. **时间范围配置**：`date_or_time_format` 配置决定了对账时使用精确到日期还是精确到时间。

3. **支付类型匹配**：支付类型通过语言包的 `lang()` 函数匹配，多语言环境下需要确保配置正确。

4. **无强校验**：系统自动计算金额后，用户仍可手动修改所有 `closed_amount_*` 字段，系统不会强制校验计算结果与实际一致。

5. **软删除**：班次记录通过 `deleted` 字段软删除，而非物理删除。

6. **参数顺序不一致（重要）**：`_calculate_total()` 函数在 `getView()` 和 `postAjax_cashup_total()` 两处调用中，cash 和 due 参数的传递顺序相反。当前因加法交换律数值结果一致，但存在维护风险，详见第六章。

7. **关班重算边界清晰（重要）**：服务端自动汇总仅在四个 `closed_amount_*` 全为0时触发一次，之后所有修改均为手工维护；前端 AJAX 仅实时重算 `closed_amount_total`，不触动其他金额。修改 `open_amount_cash` 或 `transfer_amount_cash` 后，`closed_amount_cash` 不会自动联动更新，详见第七章。

8. **跨班交接无自动化（重要）**：`transfer_amount_cash` 字段完全由用户手动输入，系统不执行以下任何操作：
   - 不自动读取上一班次的结余作为下一班次的开班现金默认值
   - 不校验相邻班次的 transfer 值是否互为相反数
   - 不生成配对的转移记录
   - 不在列表中高亮不匹配的转移金额
   实际操作中需在 `description` 字段中备注交接信息辅助人工核对，详见第八章。

9. **已关班次修正困难**：已保存关班数据后，若发现 `open_amount_cash` 或 `transfer_amount_cash` 有误，修改后不会触发自动重算（因为 `closed_amount_*` 不全为0），需要手动修正 `closed_amount_cash` 或先清空所有 closed_amount 字段再重新进入关班页面。
