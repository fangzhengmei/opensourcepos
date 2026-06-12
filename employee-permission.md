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

登出方法在 `app/Models/Employee.php` 的 `logout()`。

```php
session()->destroy();  // 销毁所有会话数据
```

登出入口有两处：
- `app/Controllers/Home.php` 的 `getLogout()`：销毁 session 后显式跳回 `login`
- `app/Controllers/Office.php` 的 `logout()`：只做销毁，是否跳转交给调用方处理

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

使用 `LIKE '{$permission_id}%'` **前缀匹配** grants 表：
- 若匹配 1 条 → 调用 `has_subpermissions()` 判断是否存在子权限定义
  - 无子权限定义 → true（放行）
  - 有子权限定义 → false（要求更细粒度的子权限分配）
- 若匹配 0 条 → false（无权限）
- 若匹配 >1 条 → true（存在子权限分配）

**设计意图**：对于带子权限的模块（如 sales 下有 sales_change_price、sales_delete），仅分配 sales 主权限不够，必须显式分配子权限才能通过。

这个实现把“能不能进控制器”和“菜单上显不显示”拆成了两套判断：
- 控制器访问走 `has_module_grant()`，按 `permission_id` 前缀统计 grants
- 菜单渲染走 `get_allowed_home_modules()` / `get_allowed_office_modules()`，只连接到与 `modules.module_id` 精确相等的那条权限

所以“只配了子权限、没配模块主权限”时，菜单可见性和控制器访问结果不一定完全一致，需要把两段逻辑一起看。

#### 方法 B：has_grant() — 精确权限检查

实现位于 `app/Models/Employee.php` 的 `has_grant()`。

精确匹配 `grants` 表的 `permission_id`，用于业务方法内部的二次校验。典型调用场景：

```php
// Sales.php:L85 — 只有拥有 reports_sales 权限才能查看某收据
if (!$this->employee->has_grant('reports_sales', $personId)) { ... }

// Sales.php:L1257 — 是否允许改单价
$data['change_price'] = $this->employee->has_grant('sales_change_price', ...);
```

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

### 3.5 员工修改互斥规则

实现位于 `app/Models/Employee.php` 的 `canModifyEmployee()`。

| 场景 | 允许修改 |
|------|----------|
| 修改自己 + 非管理员 | ✅ 允许 |
| 修改自己 + 自己是管理员 | ✅ 允许（需 admin 权限） |
| 修改他人 + 对方是 admin + 自己非 admin | ❌ 拒绝 |
| 修改他人 + 其余情况 | ✅ 允许 |

在 `app/Controllers/Employees.php` 的 `getView()` 和 `postSave()` 中都有二次校验。

### 3.6 各控制器 module_id 对照

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

### 4.1 三类重定向

| 触发条件 | 跳转目标 | 代码位置 |
|----------|----------|----------|
| **未登录** (session 中无 person_id) | `base_url('login')` | `app/Controllers/Secure_Controller.php` |
| **无模块权限** (has_module_grant 失败) | `base_url("no_access/{$module_id}/{$submodule_id}")` | `app/Controllers/Secure_Controller.php` |
| **手动登出** | `redirect()->to('login')` | `app/Controllers/Home.php` |

⚠️ 注意：前两类重定向使用的是原生 PHP `header("Location: ...")` + `exit()`，而非 CI4 的 `redirect()` 服务。这意味着后续代码会被硬中断，框架的 after 过滤器、toolbar 等将不再执行。

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
