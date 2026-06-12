# OSPOS 员工权限拦截机制梳理

## 一、整体架构：控制器继承链

OSPOS 的权限控制采用 **控制器继承 + 构造函数拦截** 的模式，而非 CodeIgniter 4 标准的 Filter 中间件。继承关系如下：

```
BaseController (抽象基类，框架标准)
       ↓
Secure_Controller (权限拦截核心，构造函数内完成登录检查+模块授权)
       ↓
   ┌───┴───┐
   │       │
Persons   各业务控制器 (Sales, Items, Reports, Config, Home, Office 等)
   ↓
Employees, Customers, Suppliers 等人员类控制器
```

- **BaseController**：纯框架基础类，仅加载通用 helper，无任何业务逻辑
- **Secure_Controller**：所有需要登录保护的控制器都继承此类，**构造函数是权限拦截的唯一入口**
- **Persons**：抽象类，人员管理（员工/客户/供应商）的通用逻辑封装

## 二、登录会话管理

### 2.1 登录入口与流程

入口文件在 `app/Controllers/Login.php`。

登录验证并非在 Login 控制器中直接处理，而是利用 **CodeIgniter Validation 自定义规则** 触发：

```php
// Login.php:L59
$rules = ['username' => 'required|login_check[data]'];
```

自定义验证规则实现在 `app/Config/Validation/OSPOSRules.php`。

```
login_check() 执行流程：
  1. installation_check() — 检查 PHP 扩展依赖 (bcmath/intl/gd/openssl/mbstring/curl/xml/json)
  2. Employee::login(username, password) — 校验账号密码，成功则写入 session
  3. gcaptcha_check() — 若启用了 Google reCAPTCHA，则校验人机验证
```

这里有个容易忽略的时序细节：`gcaptcha_check()` 排在 `Employee::login()` 后面，所以只要账号密码正确，`person_id` 会先写进 session；如果随后验证码失败，校验规则虽然返回错误，但这段代码本身没有把刚写入的登录态回滚掉。

### 2.2 登录逻辑与会话写入

核心方法在 `app/Models/Employee.php` 的 `login()`。

密码版本兼容策略：
- **hash_version = 1**：旧版 MD5 哈希。验证成功后自动升级为 `password_hash()` (hash_version=2) 并写库
- **hash_version = 2**：PHP 标准 `password_hash()` + `password_verify()`

**会话写入**：验证通过后仅写入一个字段
```php
$this->session->set('person_id', $row->person_id);
```

### 2.3 会话配置

配置文件在 `app/Config/Session.php`。

| 参数 | 值 | 说明 |
|------|-----|------|
| driver | `DatabaseHandler` (优先) / `FileHandler` (兜底) | 数据库不可用时自动降级到文件 |
| cookieName | `ospos_session` | 会话 Cookie 名 |
| expiration | `7200` (2小时) | 会话过期秒数 |
| matchIP | `true` | 校验 IP 一致性，防止会话劫持 |
| timeToUpdate | `300` (5分钟) | 每 5 分钟自动轮换 session_id |
| regenerateDestroy | `true` | 轮换时销毁旧 session，降低固定会话攻击风险 |
| savePath | `sessions` 表 | 数据库会话表名 |

### 2.4 登出流程

登出方法在 `app/Models/Employee.php` 的 `logout()`：销毁全部 session 数据。

**实际生效的登出入口**只有一处，但前端触发与后端响应的配合有一个常见误解：

- **入口方法**：[Home::getLogout()](file:///d:/fz/0601-1/solo-dogfeeding/code/15-opensourcepos/app/Controllers/Home.php#L31-L35)。方法名前缀 `get` 与实际请求方式**是一致的**，即 **GET 请求**。
- **前端触发**：[header.php:64](file:///d:/fz/0601-1/solo-dogfeeding/code/15-opensourcepos/app/Views/partial/header.php#L64) 中的 `anchor('home/logout', lang('Login.logout'))` 生成普通 `<a>` 链接，用户点击后浏览器发起 **GET** 请求到 `/home/logout`。
- **跳转由后端完成**：`getLogout()` 调用 `$this->employee->logout()` 销毁会话后，返回 `redirect()->to('login')`，浏览器收到 302 响应整页跳转到登录页。
- **关于 header_js.php 中的 AJAX 登出**：[header_js.php:68-80](file:///d:/fz/0601-1/solo-dogfeeding/code/15-opensourcepos/app/Views/partial/header_js.php#L68-L80) 中虽然有 `$("#logout").click()` 绑定 POST 请求的代码，但 `anchor()` 函数调用时**没有传入 `id` 属性**，生成的 `<a>` 标签没有 `id="logout"`，因此这段 JS 选择器**匹配不到任何元素**，实际并不会触发。这是一段**无效的遗留代码**。

> ⚠️ 注意：[Office::logout()](file:///d:/fz/0601-1/solo-dogfeeding/code/15-opensourcepos/app/Controllers/Office.php#L31-L36) 虽然也存在，但它：
> 1. 方法名没有 HTTP 动词前缀（`get`/`post`），在 `autoRoutesImproved = true` 配置下无法通过 URL 直接访问
> 2. 只调用 `$this->employee->logout()`，**没有重定向**
> 3. **没有任何视图或链接指向它**（全局搜索 `office/logout` 无匹配）
> 
> 属于完全无效的遗留代码，实际不会被用户触发。

---

### 2.5 验证码失败后的登录态残留

[OSPOSRules::login_check()](file:///d:/fz/0601-1/solo-dogfeeding/code/15-opensourcepos/app/Config/Validation/OSPOSRules.php#L29-L61) 中，`Employee::login()` 在 `gcaptcha_check()` 之前执行。这意味着：
- 账号密码正确 → `person_id` 已写入 session
- 随后验证码失败 → 整个规则返回 false，登录页重新渲染并显示错误
- 但 session 中已写入的 `person_id` **并未被清理**

如果用户此时手动访问任意受保护页面（例如直接输入 `/home`），`Secure_Controller` 会认为其已登录并放行。这个时序问题属于实现上的一个漏洞边界。

## 三、模块授权机制

### 3.1 核心拦截入口

所有受保护控制器在 `__construct()` 时调用 `parent::__construct($module_id, $submodule_id, $menu_group)`，入口在 `app/Controllers/Secure_Controller.php`。

**拦截执行顺序**：

```
第1关：is_logged_in() 检查
  └─ 未登录 → header("Location: /login") + exit()

第2关：has_module_grant() 模块授权检查
  ├─ 检查 $module_id 授权
  ├─ 若有 $submodule_id，额外检查子模块
  └─ 任一不通过 → header("Location: /no_access/{$module_id}/{$submodule_id}") + exit()

第3关：加载全局视图数据 (menu_group, allowed_modules, user_info)
```

### 3.2 数据库表结构

权限体系涉及 3 张核心表：

| 表名 | 关键字段 | 作用 |
|------|----------|------|
| `modules` | module_id, name_lang_key, desc_lang_key, sort | 模块元信息定义（sort=0 表示不在菜单显示） |
| `permissions` | permission_id, module_id | 权限定义（模块级 + 子权限级） |
| `grants` | person_id, permission_id, menu_group | **员工-权限 关联表（授权存储）** |

**menu_group 枚举**：
- `home`：仅在主菜单显示
- `office`：仅在后台（Office）菜单显示
- `both`：两边都显示

### 3.3 授权判定方法详解

#### 方法 A：has_module_grant() — 控制器级拦截

实现位于 `app/Models/Employee.php` 的 `has_module_grant()`。

这是整个权限体系中**最容易理解错**的方法，必须结合 `permissions` 和 `grants` 两张表、并注意 `has_subpermissions()` 的**反向布尔语义**才能读对。

**背景前提**：
- `permissions` 表同时存放**主权限**（permission_id 与 module_id 同名，如 `reports`、`sales`、`customers`）和**子权限**（格式为 `{主权限}_{子权限名}`，如 `reports_sales`、`sales_change_price`）
- `grants` 表给员工分配具体的 permission_id，可以是主权限也可以是子权限的任意组合

**方法实现**（Employee.php:L423-L446）：

```php
public function has_module_grant(string $permission_id, int $person_id): bool
{
    $builder = $this->db->table('grants');
    $builder->like('permission_id', $permission_id, 'after');  // 前缀匹配 grants
    $builder->where('person_id', $person_id);
    $result_count = $builder->get()->getNumRows();

    if ($result_count != 1) {
        return ($result_count != 0);   // 0条→false，≥2条→true
    }

    return $this->has_subpermissions($permission_id);
}

public function has_subpermissions(string $permission_id): bool
{
    $builder = $this->db->table('permissions');
    $builder->like('permission_id', $permission_id . '_', 'after');  // 查询有没有子权限定义

    return ($builder->get()->getNumRows() == 0);  // ⚠️ 语义反向：true=没有子权限定义，false=有子权限定义
}
```

**完整放行判定表**：

| grants 前缀匹配数 | 对应场景 | permissions 是否存在子权限定义 | 最终结果 |
|---|---|---|---|
| **0 条** | 员工没有任何与该模块相关的 grant | — | ❌ `false`（拦截） |
| **1 条** | 恰好分配 1 条相关 grant（可能是主权限，也可能是某条子权限） | 存在子权限定义（如 reports、sales） | ❌ `false`（拦截） |
| **1 条** | 恰好分配 1 条相关 grant | 不存在子权限定义（如 customers、giftcards） | ✅ `true`（放行） |
| **≥2 条** | 分配了多条相关 grant（主+子，或多个子权限） | — | ✅ `true`（放行） |

**⚠️ 关键分歧 1：`has_subpermissions()` 方法名与返回值语义完全相反。** 名字听起来是"有没有子权限"，但返回 `true` 实际表示 **permissions 表中没有该前缀的子权限定义**（行数为 0）；返回 `false` 表示**存在子权限定义**（行数不为 0）。这直接决定上表中"1 条匹配"分支的走向。

**⚠️ 关键分歧 2：对有子权限定义的模块，主权限 ≠ 放行，单条子权限 ≠ 放行。**
- 只给 `reports` 主权限 → 前缀匹配 1 条 + 存在子权限定义 → ❌ 拦截
- 只给 `reports_sales` 一条子权限 → 前缀匹配 1 条 + 存在子权限定义 → ❌ 拦截
- 给 `reports` + `reports_sales`（主+子） → 前缀匹配 ≥2 条 → ✅ 放行
- 不给 `reports` 主权限，但给 `reports_sales` + `reports_customers`（≥2 条子权限） → 前缀匹配 ≥2 条 → ✅ 也放行

**⚠️ 关键分歧 3：控制器级拦截只关心"前缀匹配数"，不区分你配的是主权限还是子权限。** 这意味着完全不分配主权限、只分配多个子权限同样可以进入控制器。但主权限还影响菜单渲染（见 3.7 节），两者是两套不同的判断逻辑。

**各模块子权限现状**（从数据库迁移脚本归纳）：

| 主模块 | 有无子权限定义 | 典型子权限 |
|---|---|---|
| `reports` | ✅ 有 | `reports_customers`, `reports_receivings`, `reports_items`, `reports_employees`, `reports_suppliers`, `reports_sales`, `reports_discounts`, `reports_taxes`, `reports_sales_taxes`, `reports_inventory`, `reports_categories`, `reports_payments`, `reports_expenses_categories` |
| `sales` | ✅ 有 | `sales_change_price`, `sales_delete` |
| `customers`, `employees`, `giftcards`, `items`, `item_kits`, `messages`, `receivings`, `config`, `suppliers`, `cashups`, `expenses`, `expenses_categories`, `taxes`, `tax_codes`, `tax_categories`, `tax_jurisdictions`, `attributes`, `office`, `home` | ❌ 无 | — |

---

### 3.7 报表控制器的双重校验

`Reports` 控制器的权限检查是**两阶段**串联执行，这是整个系统中最复杂的授权路径，也是最容易理解错的地方。

**阶段 1：父类构造函数的模块级拦截**（Secure_Controller）
- 传入 `parent::__construct('reports')`，即 `module_id = 'reports'`
- 调用 `has_module_grant('reports', $person_id)`，按 3.3 节的规则做前缀匹配统计
- 这一关只决定员工能不能进入 `/reports` 报表模块（包括报表列表页 `/reports/index`）

**阶段 2：Reports 自己构造函数的子权限精确校验**（Reports.php:L54-L94）
- 阶段 1 通过后，进入 Reports 自己的构造函数
- 从 URL 第 2 段解析出报表方法名（如 `summary_sales`、`graphical_summary_customers`、`specific_employees`）
- 用正则从方法名中提取**报表类型子模块**：

```php
$method_name = $request->getUri()->getSegment(2);           // e.g. "summary_sales"
$exploder = explode('_', $method_name);                      // ["summary", "sales"]

if (sizeof($exploder) > 1) {
    // 第1个正则：提取末尾的报表对象名（忽略 graph、row 等后缀）
    preg_match('/(?:inventory)|([^_.]*)(?:_graph|_row)?$/', $method_name, $matches);
    // 第2个正则：补上单复数（y→ies，其他加 s）
    preg_match('/^(.*?)([sy])?$/', array_pop($matches), $matches);
    $submodule_id = $matches[1] . ((count($matches) > 2) ? $matches[2] : 's');
    // e.g. "summary_sales" → submodule_id = "sales"
    //      "graphical_summary_customers" → submodule_id = "customers"
    //      "specific_employees" → submodule_id = "employees"
    //      "summary_discounts" → submodule_id = "discounts"
    //      "summary_taxes" → submodule_id = "taxes"
    //      "summary_payments" → submodule_id = "payments"
    //      "inventory_summary" → submodule_id = "inventory"  (inventory 不走复数变换)
    //      "summary_expenses_categories" → submodule_id = "categories" ⚠️

    // 精确校验 reports_{submodule_id} 子权限
    if (!$this->employee->has_grant('reports_' . $submodule_id, ...)) {
        header('Location: ' . base_url('no_access/reports/reports_' . $submodule_id));
        exit();
    }
}
```

**阶段 2 仅在 URL 第 2 段含下划线（即具体报表方法，如 `summary_sales`）时触发；**
纯列表页 `/reports/index`（第 2 段为 `index` 或空，`explode` 后长度为 1）**不会**进入阶段 2 校验。

**⚠️ 报表权限设计的特殊约束**：
1. 进入报表模块列表页只需阶段 1 通过（grants 中 reports 前缀匹配 ≥2 条，或无子权限定义——但 reports 有子权限，所以实际上必须 ≥2 条）
2. 点击具体报表进入后，阶段 2 额外校验对应的 `reports_{submodule_id}` 精确权限，缺任一子权限都会被单独弹到 `no_access`
3. 正则提取 `submodule_id` 时对 `expenses_categories` 的处理有缺陷：URL `summary_expenses_categories` 被提取成 `"categories"`（匹配最后一个下划线后的词），实际校验的是 `reports_categories` 而非 `reports_expenses_categories`。不过 `permissions` 表中**同时**存在 `reports_categories` 和 `reports_expenses_categories` 两条记录，所以此处是否与实际授权对应要结合数据看。

---

### 3.8 Sales 等模块的方法级细粒度校验

除了报表的"构造函数两阶段"模式，其他带子权限的模块（如 `Sales`）采用**构造函数先过模块级、具体方法内再做精确检查**的分散模式：

| 位置 | 校验方法 | 权限 ID | 作用 |
|---|---|---|---|
| Sales 构造函数 | `has_module_grant('sales')` | 前缀匹配 | 决定能否进入 Sales 模块 |
| [Sales.php:L85](file:///d:/fz/0601-1/solo-dogfeeding/code/15-opensourcepos/app/Controllers/Sales.php#L85) | `has_grant('reports_sales', $personId)` | `reports_sales` | 查看销售收据/发票前二次校验 |
| [Sales.php:L1257](file:///d:/fz/0601-1/solo-dogfeeding/code/15-opensourcepos/app/Controllers/Sales.php#L1257) | `has_grant('sales_change_price', ...)` | `sales_change_price` | 销售界面是否允许改单价 |
| [Sales.php:L1391](file:///d:/fz/0601-1/solo-dogfeeding/code/15-opensourcepos/app/Controllers/Sales.php#L1391) | `has_grant('sales_delete', $employee_id)` | `sales_delete` | 是否允许删除销售单 |
| [Reports.php:L87](file:///d:/fz/0601-1/solo-dogfeeding/code/15-opensourcepos/app/Controllers/Reports.php#L87) | `has_grant('reports_' . $submodule_id, ...)` | `reports_*` | 具体报表子权限 |
| [Items.php:L305](file:///d:/fz/0601-1/solo-dogfeeding/code/15-opensourcepos/app/Controllers/Items.php#L305) | `has_grant('item_kits', ...)` | `item_kits` | 商品管理中是否禁用商品套件功能 |
| [Receivings.php:L247](file:///d:/fz/0601-1/solo-dogfeeding/code/15-opensourcepos/app/Controllers/Receivings.php#L247) | `has_grant('employees', ...)` | `employees` | 采购单能否指派员工 |

---

### 3.9 菜单渲染与控制器放行的两套独立逻辑

"菜单上显不显示"和"能不能进控制器"是两套独立判断，结果可能不一致：

- **控制器访问**（`Secure_Controller::__construct`）：走 `has_module_grant()`，对 `grants` 做 `permission_id` 前缀统计
- **菜单渲染**（`Module::get_allowed_home_modules()` / `get_allowed_office_modules()`）：`modules` 表 JOIN `permissions`（精确等于 `module_id`）JOIN `grants`，要求必须存在**与 module_id 精确相等**的那条 grant

这意味着：
- ✅ 只配主权限 `reports`（无子权限） → 菜单显示报表入口，但进不去具体报表（阶段 1 前缀匹配 1 条 + 有子权限定义 → 被拦）
- ✅ 只配 `reports_sales` + `reports_customers`（两条子权限，不配主权限） → 阶段 1 前缀匹配 2 条 → 能进报表控制器并打开列表页，也能打开 sales/customers 具体报表，但**菜单上不显示报表模块**（因为没有 `reports` 主权限的 grant，JOIN 查不出来）
- ✅ 配 `reports` 主权限 + 任意 ≥1 条子权限 → 菜单显示 + 能进控制器 + 阶段 2 按配置的子权限精确放行

这也是管理员分配权限时最容易踩坑的地方。

### 3.4 管理员判定

实现位于 `app/Models/Employee.php` 的 `isAdmin()`。

```
判定逻辑：
  person_id === 1 → 超级管理员（硬编码）
  否则遍历 ADMIN_MODULES 常量，必须拥有全部模块权限
```

`ADMIN_MODULES` 常量在 `app/Config/Constants.php`：
```php
['customers', 'employees', 'giftcards', 'items', 'item_kits', 'messages',
 'receivings', 'reports', 'sales', 'config', 'suppliers']
```

### 3.11 员工修改互斥规则

实现位于 `app/Models/Employee.php` 的 `canModifyEmployee()`。

| 场景 | 允许修改 |
|------|----------|
| 修改自己 + 非管理员 | ✅ 允许 |
| 修改自己 + 自己是管理员 | ✅ 允许（需 admin 权限） |
| 修改他人 + 对方是 admin + 自己非 admin | ❌ 拒绝 |
| 修改他人 + 其余情况 | ✅ 允许 |

在 `app/Controllers/Employees.php` 的 `getView()` 和 `postSave()` 中都有二次校验。

### 3.12 各控制器 module_id 对照

| 控制器 | module_id | 说明 |
|--------|-----------|------|
| Home | `home` | 首页，menu_group='home' |
| Office | `office` | 后台，menu_group='office' |
| Sales | `sales` | 销售 |
| Receivings | `receivings` | 采购/收货 |
| Items | `items` | 商品 |
| Item_kits | `item_kits` | 商品套件 |
| Employees | `employees` | 员工管理（通过 Persons 继承） |
| Customers | `customers` | 客户管理（通过 Persons 继承） |
| Suppliers | `suppliers` | 供应商管理（通过 Persons 继承） |
| Reports | `reports` | 报表（子模块动态校验 reports_sales / reports_customers 等） |
| Config | `config` | 系统配置 |
| Giftcards | `giftcards` | 礼品卡 |
| Cashups | `cashups` | 收银结算 |
| Expenses | `expenses` | 支出 |
| Expenses_categories | `expenses_categories` | 支出分类 |
| Taxes | `taxes` | 税种 |
| Tax_codes / Tax_categories / Tax_jurisdictions | 对应模块名 | 税配置子模块 |
| Attributes | `attributes` | 商品属性 |
| Messages | `messages` | 消息通知 |

## 四、重定向与异常页面

### 4.1 四类重定向

| 类型 | 触发条件 | 跳转目标 | 代码位置 |
|------|----------|----------|----------|
| **未登录** | session 中无 `person_id` | `base_url('login')` | [Secure_Controller.php:L42-L45](file:///d:/fz/0601-1/solo-dogfeeding/code/15-opensourcepos/app/Controllers/Secure_Controller.php#L42-L45) |
| **无模块权限** | `has_module_grant()` 失败 | `base_url("no_access/{$module_id}/{$submodule_id}")` | [Secure_Controller.php:L48-L54](file:///d:/fz/0601-1/solo-dogfeeding/code/15-opensourcepos/app/Controllers/Secure_Controller.php#L48-L54) |
| **无报表子权限** | Reports 阶段 2 校验失败 | `base_url("no_access/reports/reports_{$submodule_id}")` | [Reports.php:L86-L90](file:///d:/fz/0601-1/solo-dogfeeding/code/15-opensourcepos/app/Controllers/Reports.php#L86-L90) |
| **手动登出** | 点击退出链接 | `redirect()->to('login')` | [Home.php:L31-L35](file:///d:/fz/0601-1/solo-dogfeeding/code/15-opensourcepos/app/Controllers/Home.php#L31-L35) |

⚠️ 注意：前**三类**重定向（未登录、无模块权限、无报表子权限）使用的是原生 PHP `header("Location: ...")` + `exit()`，而非 CI4 的 `redirect()` 服务。这意味着后续代码会被硬中断，框架的 after 过滤器、toolbar 等将不再执行。只有**手动登出**使用了 CI4 的 `redirect()` 服务。

### 4.2 无权限页面

无权限页控制器在 `app/Controllers/No_access.php`。

- 继承 `BaseController`（非 Secure_Controller），避免死循环重定向
- 接收 `module_id` 和 `permission_id` 两个参数，用于在视图中提示具体缺少的权限
- 对应路由在 `app/Config/Routes.php` 注册

### 4.3 会话过期的隐式重定向

当 session 超过 7200 秒过期后，session 中的 `person_id` 自动失效。下一次请求任意受保护控制器时：

1. `Secure_Controller::__construct()` 第1关判定 `is_logged_in() === false`
2. 执行 `header("Location:" . base_url('login')); exit();`
3. 用户跳回登录页，无需显式的过期提示

## 五、全局视图数据与菜单渲染

通过权限校验后，Secure_Controller 会把菜单数据写入全局视图：

```php
// 根据 menu_group (home/office) 获取允许的模块
$allowed_modules = $menu_group == 'home'
    ? $this->module->get_allowed_home_modules(...)   // menu_group in ('home', 'both')
    : $this->module->get_allowed_office_modules(...); // menu_group in ('office', 'both')

// 视图中可直接使用：
// $allowed_modules — 允许访问的模块列表（用于渲染侧边栏菜单）
// $user_info       — 当前登录员工完整信息
// $controller_name — 当前 module_id（用于高亮当前菜单）
```

`menu_group` 还会被写回 session：进入 `Home` 时固定成 `home`，进入 `Office` 时固定成 `office`。这意味着同一个登录态下，控制器不仅决定“能不能进”，还顺带决定接下来菜单按哪个分组渲染。

## 六、CSRF 与过滤器补充

虽然权限校验不使用 CI4 Filter，但 `app/Config/Filters.php` 里仍配置了全局安全防护：

| 过滤器 | 作用 | 例外 |
|--------|------|------|
| `csrf` | 全局 CSRF 令牌校验 | `login`、`migrate` 路由（登录页 POST 不能被 CSRF 拦截） |
| `honeypot` | 反垃圾表单蜜罐 | - |
| `invalidchars` | 非法字符检测 | - |
| `secureheaders` | 安全响应头 | - |
| `forcehttps` | 强制 HTTPS | - |
