# OSPOS 报表路由匹配规则与参数绑定机制分析（基于 CI4 实际行为）

本文档深入解析 OSPOS 报表系统的路由匹配规则、`(:any)` 任意段占位符的行为、多段参数开关的影响，以及三条核心路径（通用汇总、折扣报表、仅按日期统计）的实际命中过程与参数流转。

---

## 一、框架级关键配置

所有分析基于 OSPOS 在 [Routing.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Config/Routing.php) 中的实际配置：

| 配置项 | 值 | 影响 |
|--------|----|------|
| `$autoRoute` (L97) | `true` | 未命中定义路由时，按 `段1/段2(方法)/段3(参数1)/段4(参数2)/...` 模式自动路由 |
| `$multipleSegmentsOneParam` (L123) | **`false`** | `(:any)` 捕获的多段值（含 `/`）会被**按 `/` 再分割**成多个参数传递 |
| `$prioritize` (L115) | `false` | 路由按**定义顺序**从上到下匹配，不按优先级排序 |
| `$defaultController` (L51) | `Login` | 无匹配时的默认控制器 |
| `$defaultMethod` (L60) | `index` | 方法缺失时的默认方法 |

### 1.1 `$multipleSegmentsOneParam = false` 的核心影响

**这是决定本系统路由行为的最关键开关。**

| 配置值 | `(:any)` 含 `/` 时的参数传递 |
|--------|----------------------------|
| `true`（CI4.5+ 默认） | `(:any)` 捕获的 `a/b/c` 作为**单个整体参数**传递 → `method("a/b/c")` |
| **`false`（OSPOS 当前配置）** | `(:any)` 捕获的 `a/b/c` **按 `/` 再分割**成多个参数 → `method("a", "b", "c")` |

OSPOS 当前使用 `false`，所以**任何包含 `/` 的占位符捕获值都会导致参数数量不可预测地增加**。

### 1.2 `(:any)` 占位符的段数匹配行为

根据 CodeIgniter 4 官方文档，`(:any)` 的行为取决于它在路由模式中的位置：

| `(:any)` 的位置 | 匹配行为 |
|-----------------|---------|
| **在路由末尾**（后面没有字面段或其他占位符） | 可以匹配**从该位置到 URL 末尾的所有字符**，包括 `/`，即**跨越多个段** |
| **在路由中间**（后面有字面段或其他占位符） | 只能匹配**单个段**（不含 `/`），因为后面还有内容需要匹配 |

**⚠️ 官方警告：不要在 `(:any)` 后面放置任何占位符！** 因为 `(:any)` 在末尾时匹配的段数不确定，会导致参数数量变化。

---

## 二、Routes.php 中每条报表路由的逐条分析

路由定义在 [Routes.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-opensourcepos/app/Config/Routes.php#L18-L42)，按定义顺序（匹配优先级）从上到下分析：

### 路由 #1：`reports/summary_(:any)/(:any)/(:any)` （L18）

```php
$routes->add('reports/summary_(:any)/(:any)/(:any)', 'Reports::Summary_$1/$2/$3/$4');
```

| 项目 | 分析 |
|------|------|
| **按 `/` 分割单元** | `['reports', 'summary_(:any)', '(:any)', '(:any)']` → 共 4 个单元 |
| **单元 1** | `reports`（字面量，匹配段 1） |
| **单元 2** | `summary_(:any)`（字面+第 1 个占位符，后面还有 2 个单元 → **只能匹配单个段**，匹配段 2 中 `_` 后的部分） |
| **单元 3** | `(:any)`（第 2 个占位符，后面还有 1 个单元 → **只能匹配单个段**，匹配段 3） |
| **单元 4** | `(:any)`（第 3 个占位符，**在路由末尾！可以匹配 1 个或多个段**，匹配段 4 到 URL 末尾的所有段） |
| **匹配的 URL 段数** | **4 段或更多**（单元 1+2+3 各占 1 段 + 单元 4 占 1~N 段） |
| **动态方法名** | `Summary_$1`，其中 `$1` = 单元 2 中 `summary_` 后的部分 |
| **反向引用** | `$1/$2/$3/$4` → **有 3 个占位符但有 4 个反向引用！`$4` 悬空，永远是空串！** |
| **参数传递（OSPOS 配置 `$multipleSegmentsOneParam=false`）** | 先替换 `$1~$4`，再按 `/` 分割整个参数部分，得到方法实参 |

#### 不同 URL 段数下的匹配与参数示例

URL 结构：`reports / summary_sales / 段3 / 段4 / 段5 / 段6 / 段7`

| 实际 URL | URL 段数 | 是否匹配路由 #1？ | `$1` | `$2` | `$3`（单元4，末尾） | `$4` | 目标路径 | 按 `/` 分割后的实参 |
|---------|---------|------------------|------|------|-------------------|------|---------|-------------------|
| `reports/summary_sales/a/b` | 4 | ✅ 匹配 | `sales` | `a` | `b`（1段） | `''` | `Summary_sales/a/b/` | `['a', 'b', '']`（3个） |
| `reports/summary_sales/a/b/c` | 5 | ✅ 匹配 | `sales` | `a` | `b/c`（2段） | `''` | `Summary_sales/a/b/c/` | `['a', 'b', 'c', '']`（4个） |
| `reports/summary_sales/a/b/c/d` | 6 | ✅ 匹配 | `sales` | `a` | `b/c/d`（3段） | `''` | `Summary_sales/a/b/c/d/` | `['a', 'b', 'c', 'd', '']`（5个） |
| `reports/summary_sales/a/b/c/d/e` | 7 | ✅ 匹配 | `sales` | `a` | `b/c/d/e`（4段） | `''` | `Summary_sales/a/b/c/d/e/` | `['a', 'b', 'c', 'd', 'e', '']`（6个） |

**关键发现：7 段 URL（前端实际生成的带参 URL）确实会命中路由 #1，而不是走自动路由！**

---

### 路由 #2：`reports/summary_expenses_categories` （L19）

```php
$routes->add('reports/summary_expenses_categories', 'Reports::date_input_only');
```

| 项目 | 分析 |
|------|------|
| **按 `/` 分割单元** | `['reports', 'summary_expenses_categories']` → 2 个单元，全部字面量 |
| **匹配的 URL 段数** | **精确等于 2 段**（无任何占位符，段数必须严格相等） |
| **调用方法** | `Reports::date_input_only()` → 无参数 |
| **优先级** | 定义在路由 #5（通配 `summary_(:any)`）之前，所以同是 2 段 URL 时优先命中 |

---

### 路由 #3：`reports/summary_payments` （L20）

```php
$routes->add('reports/summary_payments', 'Reports::date_input_only');
```

| 项目 | 分析 |
|------|------|
| **单元** | 2 个字面量单元 |
| **匹配的 URL 段数** | 精确等于 2 段 |
| **调用方法** | `Reports::date_input_only()` → 无参数 |

---

### 路由 #4：`reports/summary_discounts` （L21）

```php
$routes->add('reports/summary_discounts', 'Reports::summary_discounts_input');
```

| 项目 | 分析 |
|------|------|
| **单元** | 2 个字面量单元 |
| **匹配的 URL 段数** | 精确等于 2 段 |
| **调用方法** | `Reports::summary_discounts_input()` → 注入了折扣类型下拉选项 |

---

### 路由 #5：`reports/summary_(:any)` （L22）

```php
$routes->add('reports/summary_(:any)', 'Reports::date_input');
```

| 项目 | 分析 |
|------|------|
| **按 `/` 分割单元** | `['reports', 'summary_(:any)']` → 2 个单元 |
| **单元 2** | `summary_(:any)`（第 1 个占位符，**在路由末尾！可以匹配 1 个或多个段**） |
| **匹配的 URL 段数** | **2 段或更多**（但注意：2 段 URL 已被路由 #2/#3/#4 的字面量优先匹配了） |
| **不使用 `$1`** | 路由目标中没有使用反向引用 `$1`，所以不管 `$1` 捕获了什么，都不会作为参数传递 |
| **调用方法** | `Reports::date_input()`（固定方法，不根据 `$1` 动态变化） |

**重要：** 当 URL 是 3 段（如 `reports/summary_sales/foo`）时，路由 #5 也会匹配，因为末尾的 `(:any)` 可以跨越多个段，捕获 `sales/foo`。

---

### 路由 #6：`reports/graphical_(:any)/(:any)/(:any)` （L24）

```php
$routes->add('reports/graphical_(:any)/(:any)/(:any)', 'Reports::Graphical_$1/$2/$3/$4');
```

与路由 #1 结构完全相同，只是前缀变成了 `graphical_`：

| 项目 | 分析 |
|------|------|
| **末尾 `(:any)`** | 单元 4 在末尾，可以跨越段匹配 |
| **匹配 URL 段数** | 4 段或更多 |
| **`$4` 悬空** | 同样存在，永远是空串 |

---

### 路由 #7：`reports/graphical_summary_expenses_categories` （L25）

字面量路由，精确 2 段匹配 → `date_input_only()`

### 路由 #8：`reports/graphical_summary_discounts` （L26）

字面量路由，精确 2 段匹配 → `summary_discounts_input()`

### 路由 #9：`reports/graphical_(:any)` （L27）

末尾 `(:any)`，2 段或更多匹配 → `Reports::date_input()`（不使用 `$1`）

### 路由 #10：`reports/inventory_(:any)/(:any)` （L29）

```php
$routes->add('reports/inventory_(:any)/(:any)', 'Reports::Inventory_$1/$2');
```

| 项目 | 分析 |
|------|------|
| **按 `/` 分割单元** | `['reports', 'inventory_(:any)', '(:any)']` → 3 个单元 |
| **单元 2** | `inventory_(:any)`（后面还有 1 个单元 → 只能匹配单个段，段 2 中 `_` 后部分） |
| **单元 3** | `(:any)`（**在路由末尾！可以匹配 1 个或多个段**） |
| **匹配 URL 段数** | 3 段或更多 |
| **反向引用** | `$1/$2`（2 个占位符对应 2 个引用，**无悬空问题**） |

---

### 路由 #11：`reports/inventory_low` （L30）

字面量路由，精确 2 段 → `inventory_low()`

### 路由 #12：`reports/inventory_summary` （L31）

字面量路由，精确 2 段 → `inventory_summary_input()`

### 路由 #13：`reports/inventory_summary/(:any)/(:any)/(:any)` （L32）

```php
$routes->add('reports/inventory_summary/(:any)/(:any)/(:any)', 'Reports::inventory_summary/$1/$2/$3');
```

| 项目 | 分析 |
|------|------|
| **按 `/` 分割单元** | `['reports', 'inventory_summary', '(:any)', '(:any)', '(:any)']` → 5 个单元 |
| **前两个单元** | 字面量（段 1、段 2） |
| **单元 5** | `(:any)` 在末尾 → 可以跨越段 |
| **匹配 URL 段数** | 5 段或更多 |
| **反向引用** | 3 个占位符对应 3 个引用 → **无悬空问题** |

---

### 路由 #14：`reports/detailed_(:any)/(:any)/(:any)/(:any)` （L34）

```php
$routes->add('reports/detailed_(:any)/(:any)/(:any)/(:any)', 'Reports::Detailed_$1/$2/$3/$4');
```

| 项目 | 分析 |
|------|------|
| **按 `/` 分割单元** | 5 个单元：`['reports', 'detailed_(:any)', '(:any)', '(:any)', '(:any)']` |
| **单元 5** | `(:any)` 在末尾 → 可以跨越段 |
| **匹配 URL 段数** | 5 段或更多 |
| **反向引用** | 4 个占位符对应 4 个引用 → **无悬空问题**（和路由 #13 一样正确） |

---

### 路由 #15：`reports/detailed_sales` （L35）

字面量路由，精确 2 段 → `date_input_sales()`

### 路由 #16：`reports/detailed_receivings` （L36）

字面量路由，精确 2 段 → `date_input_recv()`

### 路由 #17：`reports/specific_(:any)/(:any)/(:any)/(:any)` （L38）

```php
$routes->add('reports/specific_(:any)/(:any)/(:any)/(:any)', 'Reports::Specific_$1/$2/$3/$4');
```

结构与路由 #14 完全相同：
- 5 个单元，末尾 `(:any)` 可以跨越段
- 匹配 5 段或更多
- 4 个占位符对应 4 个反向引用 → **无悬空问题**

### 路由 #18-#21：`reports/specific_*` （L39-L42）

4 条字面量路由，精确 2 段匹配：
- `specific_customers` → `specific_customer_input()`
- `specific_employees` → `specific_employee_input()`
- `specific_discounts` → `specific_discount_input()`
- `specific_suppliers` → `specific_supplier_input()`

---

## 三、路由匹配优先级总表（按定义顺序）

```
高  ↓  1. reports/summary_(:any)/(:any)/(:any)        ← 4段+ 通配
优  ↓  2. reports/summary_expenses_categories         ← 字面2段
先  ↓  3. reports/summary_payments                    ← 字面2段
级  ↓  4. reports/summary_discounts                   ← 字面2段
    ↓  5. reports/summary_(:any)                      ← 2段+ 通配
    ↓  6. reports/graphical_(:any)/(:any)/(:any)      ← 4段+ 通配
    ↓  7. reports/graphical_summary_expenses_categories ← 字面2段
    ↓  8. reports/graphical_summary_discounts         ← 字面2段
    ↓  9. reports/graphical_(:any)                    ← 2段+ 通配
    ↓ 10. reports/inventory_(:any)/(:any)             ← 3段+ 通配
    ↓ 11. reports/inventory_low                       ← 字面2段
    ↓ 12. reports/inventory_summary                   ← 字面2段
    ↓ 13. reports/inventory_summary/(:any)/(:any)/(:any) ← 5段+ 通配
    ↓ 14. reports/detailed_(:any)/(:any)/(:any)/(:any) ← 5段+ 通配
    ↓ 15. reports/detailed_sales                      ← 字面2段
    ↓ 16. reports/detailed_receivings                 ← 字面2段
    ↓ 17. reports/specific_(:any)/(:any)/(:any)/(:any) ← 5段+ 通配
低  ↓ 18-21. reports/specific_* 四条字面量             ← 字面2段
```

---

## 四、三条核心路径的完整匹配过程

### 4.1 路径 A：通用汇总报表（以 `summary_sales` 为例）

#### 场景 1：访问输入表单（2 段 URL）

```
URL: reports/summary_sales（2段）

匹配过程（从上到下）：
  1. 路由 #1: 需要 4段+ → ❌ 段数不够
  2. 路由 #2: "summary_expenses_categories" ≠ "summary_sales" → ❌
  3. 路由 #3: "summary_payments" ≠ "summary_sales" → ❌
  4. 路由 #4: "summary_discounts" ≠ "summary_sales" → ❌
  5. 路由 #5: reports/summary_(:any)（2段+） → ✅ 匹配！
     $1 = "sales"
     目标: Reports::date_input（不使用 $1）
     调用: Reports::date_input()（无参数）

结果：渲染日期选择表单（注入了销售类型和仓库下拉选项）
```

#### 场景 2：提交表单（7 段 URL，前端 JS 拼 5 个参数）

```
URL: reports/summary_sales/2024-01-01/2024-12-31/complete/all/0（7段）
段:  1-reports  2-summary_sales  3-2024-01-01  4-2024-12-31  5-complete  6-all  7-0

匹配过程（从上到下）：
  1. 路由 #1: reports/summary_(:any)/(:any)/(:any)（4段+） → ✅ 匹配！
     单元1 → 段1: reports
     单元2 → 段2: summary_sales → $1 = "sales"
     单元3 → 段3: 2024-01-01 → $2 = "2024-01-01"
     单元4（末尾）→ 段4+段5+段6+段7: "2024-12-31/complete/all/0" → $3 = "2024-12-31/complete/all/0"
     $4 = ""（悬空，无对应占位符）

     替换目标: Reports::Summary_sales/2024-01-01/2024-12-31/complete/all/0/
     参数部分（按 / 分割）: ["2024-01-01", "2024-12-31", "complete", "all", "0", ""]（6个参数）

     方法签名: summary_sales($start_date, $end_date, $sale_type, $location_id = 'all')
     参数绑定:
       $start_date  = "2024-01-01"  ← ✅ 正确（段3）
       $end_date    = "2024-12-31"  ← ✅ 正确（段4）
       $sale_type   = "complete"    ← ✅ 正确（段5）
       $location_id = "all"         ← ✅ 正确（段6）
       多余参数 "0"（段7-discount_type）和 ""（悬空 $4）被 PHP 静默忽略！

结果：通过路由 #1 命中 Summary_sales()，参数绑定完全正确！
```

#### 场景 3：5 段 URL（用户只选了 3 个参数，没有 location_id）

```
URL: reports/summary_sales/2024-01-01/2024-12-31/complete（5段）

匹配：路由 #1 → ✅
  $1 = "sales"
  $2 = "2024-01-01"
  $3 = "2024-12-31/complete"
  $4 = ""
  参数分割: ["2024-01-01", "2024-12-31", "complete", ""]（4个）
  绑定:
    $start_date  = "2024-01-01" ✅
    $end_date    = "2024-12-31" ✅
    $sale_type   = "complete"   ✅
    $location_id = ""           ← ⚠️ 收到空串，而不是默认值 "all"！

  ⚠️  BUG！当 location_id 未显式传递时，路由 #1 会传空串而非默认值。
  但前端 JS 总是拼 5 个参数（含 location_id='all'），所以实际几乎不触发。
```

| 场景 | URL 段数 | 命中路由 | 调用方法 | 参数正确性 |
|------|---------|---------|---------|-----------|
| 输入表单 | 2 段 | #5 | `date_input()` | ✅ |
| 完整参数 | 7 段 | **#1** | `Summary_sales(6个参数，末尾2个忽略)` | ✅ 完全正确 |
| 缺 location_id | 5 段 | #1 | `Summary_sales(4个参数)` | ⚠️ $location_id 收到空串 |

---

### 4.2 路径 B：折扣报表（`summary_discounts` 和 `specific_discounts`）

#### 路径 B-1：`summary_discounts` 折扣汇总

**场景 1：输入表单（2 段 URL）**

```
URL: reports/summary_discounts（2段）

匹配：
  1. 路由 #1: 4段+ → ❌
  2-3. 字面量不匹配 → ❌
  4. 路由 #4: "summary_discounts" 字面量精确匹配 → ✅
  调用: Reports::summary_discounts_input()（注入了折扣类型下拉选项）
```

**场景 2：完整参数（7 段 URL）**

```
URL: reports/summary_discounts/2024-01-01/2024-12-31/complete/all/0（7段）

匹配：
  1. 路由 #1: reports/summary_(:any)/(:any)/(:any)（4段+） → ✅ 匹配！
     （注意：summary_discounts 满足 summary_ 前缀模式！）
     $1 = "discounts"
     $2 = "2024-01-01"
     $3 = "2024-12-31/complete/all/0"
     $4 = ""
     动态方法名: Summary_discounts
     参数分割: ["2024-01-01", "2024-12-31", "complete", "all", "0", ""]（6个）

  方法签名: summary_discounts($start_date, $end_date, $sale_type, $location_id='all', $discount_type=0)
  绑定:
    $start_date    = "2024-01-01"  ✅
    $end_date      = "2024-12-31"  ✅
    $sale_type     = "complete"    ✅
    $location_id   = "all"         ✅
    $discount_type = "0"           ✅
    多余 "" 被忽略

结果：通过路由 #1 命中 Summary_discounts()，参数完全正确！
（路由 #4 的字面量只匹配 2 段，7 段 URL 不匹配 #4）
```

**关键发现：折扣汇总 `summary_discounts` 的带参请求实际上也命中了通用汇总路由 #1！** 因为 `summary_discounts` 满足 `summary_xxx` 的通用前缀模式。

#### 路径 B-2：`specific_discounts` 折扣明细

**场景 1：输入表单（2 段 URL）**

```
URL: reports/specific_discounts（2段）

匹配：
  1-16. 前面路由段数或字面量不匹配 → ❌
  17. 路由 #17: 需要 5段+ → ❌ 段数不够
  18-19. specific_customers/employees → ❌
  20. 路由 #20: "specific_discounts" 字面量精确匹配 → ✅
  调用: Reports::specific_discount_input()
```

**场景 2：完整参数（7 段 URL）**

```
URL: reports/specific_discounts/2024-01-01/2024-12-31/10/complete/0（7段）
段:  1-reports  2-specific_discounts  3-2024-01-01  4-2024-12-31  5-10(discount值)  6-complete  7-0(折扣类型)

匹配：
  1-16. 前面路由前缀不匹配 → ❌
  17. 路由 #17: reports/specific_(:any)/(:any)/(:any)/(:any)（5段+） → ✅ 匹配！
      单元1 → 段1: reports
      单元2 → 段2: specific_discounts → $1 = "discounts"
      单元3 → 段3: 2024-01-01 → $2 = "2024-01-01"
      单元4 → 段4: 2024-12-31 → $3 = "2024-12-31"
      单元5（末尾）→ 段5+段6+段7: "10/complete/0" → $4 = "10/complete/0"

      替换目标: Reports::Specific_discounts/2024-01-01/2024-12-31/10/complete/0
      参数分割: ["2024-01-01", "2024-12-31", "10", "complete", "0"]（5个）

      方法签名: specific_discounts($start_date, $end_date, $discount, $sale_type, $discount_type)
      绑定:
        $start_date    = "2024-01-01"  ✅（段3）
        $end_date      = "2024-12-31"  ✅（段4）
        $discount      = "10"          ✅（段5-折扣值！）
        $sale_type     = "complete"    ✅（段6）
        $discount_type = "0"           ✅（段7）
      形参数=实参数，完全匹配！无多余参数！

结果：通过路由 #17 命中 Specific_discounts()，参数完全正确！
```

---

### 4.3 路径 C：仅按日期统计（`summary_payments` 和 `summary_expenses_categories`）

#### 路径 C-1：`summary_payments` 支付方式统计

**场景 1：输入表单（2 段 URL）**

```
URL: reports/summary_payments（2段）

匹配：
  1. 路由 #1: 4段+ → ❌
  2. summary_expenses_categories → ❌
  3. 路由 #3: "summary_payments" 字面量精确匹配 → ✅
  调用: Reports::date_input_only()（无注入 $mode/$stock_locations）
        → 表单上没有销售类型/仓库下拉选项
```

**场景 2：提交表单（7 段 URL，前端 JS 硬拼 5 个参数）**

```
URL: reports/summary_payments/2024-01-01/2024-12-31/0/all/0（7段）
段:  1-reports  2-summary_payments  3-2024-01-01  4-2024-12-31  5-0  6-all  7-0
（注意：段5-7 是前端 JS 强行拼的假值，因为 date_input_only 没注入这些选项，所以值为 0/all/0）

匹配：
  1. 路由 #1: reports/summary_(:any)/(:any)/(:any)（4段+） → ✅ 匹配！
     （summary_payments 满足 summary_ 前缀！）
     $1 = "payments"
     $2 = "2024-01-01"
     $3 = "2024-12-31/0/all/0"
     $4 = ""
     动态方法名: Summary_payments
     参数分割: ["2024-01-01", "2024-12-31", "0", "all", "0", ""]（6个参数）

  方法签名: summary_payments($start_date, $end_date)（只有 2 个形参！）
  绑定:
    $start_date = "2024-01-01"  ✅（段3）
    $end_date   = "2024-12-31"  ✅（段4）
    多余参数 "0", "all", "0", "" 共 4 个被 PHP 静默忽略！

结果：通过路由 #1 命中 Summary_payments()，前 2 个参数正确，后 4 个假值被忽略。
控制器层硬编码 sale_type='complete', location_id='all'，所以假值不影响。
```

**关键发现：仅按日期统计的带参请求同样命中了通用汇总路由 #1！** 只是因为方法形参数少，多余假值参数被截断忽略。

#### 路径 C-2：`summary_expenses_categories` 费用分类统计

**场景 1：输入表单（2 段 URL）**

```
URL: reports/summary_expenses_categories（2段）

匹配：
  1. 路由 #1: 4段+ → ❌
  2. 路由 #2: "summary_expenses_categories" 字面量精确匹配 → ✅
  调用: Reports::date_input_only()
```

**场景 2：提交表单（7 段 URL）**

```
URL: reports/summary_expenses_categories/2024-01-01/2024-12-31/0/all/0（7段）

匹配：
  1. 路由 #1: reports/summary_(:any)/(:any)/(:any)（4段+） → ✅ 匹配！
     （summary_expenses_categories 满足 summary_ 前缀！）
     $1 = "expenses_categories"
     $2 = "2024-01-01"
     $3 = "2024-12-31/0/all/0"
     $4 = ""
     动态方法名: Summary_expenses_categories
     参数分割: ["2024-01-01", "2024-12-31", "0", "all", "0", ""]（6个参数）

  方法签名: summary_expenses_categories($start_date, $end_date, $sale_type)（3 个形参）
  绑定:
    $start_date = "2024-01-01"  ✅（段3）
    $end_date   = "2024-12-31"  ✅（段4）
    $sale_type  = "0"           ← ⚠️ 收到非法值 "0"（段5是前端拼的假值）
    多余 "all", "0", "" 被忽略

  结果：第 3 个参数 $sale_type 收到非法值 "0"，但模型层查询完全忽略此字段，所以不影响结果。
```

---

## 五、三条路径的命中方式综合对照表

| 场景 | 路径 | URL 结构 | URL 段数 | 命中路由 | 动态方法名 | 实参数 | 形参数 | 多余参数 | 参数正确性 |
|------|------|---------|---------|---------|-----------|--------|--------|---------|-----------|
| **输入表单** | A 通用汇总（sales） | `reports/summary_sales` | 2 | #5 | —（固定 `date_input`） | 0 | 0 | 0 | ✅ |
| | B-1 折扣汇总 | `reports/summary_discounts` | 2 | #4 | —（固定 `summary_discounts_input`） | 0 | 0 | 0 | ✅ |
| | B-2 折扣明细 | `reports/specific_discounts` | 2 | #20 | —（固定 `specific_discount_input`） | 0 | 0 | 0 | ✅ |
| | C-1 支付方式 | `reports/summary_payments` | 2 | #3 | —（固定 `date_input_only`） | 0 | 0 | 0 | ✅ |
| | C-2 费用分类 | `reports/summary_expenses_categories` | 2 | #2 | —（固定 `date_input_only`） | 0 | 0 | 0 | ✅ |
| **带参数请求** | A 通用汇总（sales） | `reports/summary_sales/start/end/type/loc/disc` | 7 | **#1** | `Summary_sales` | 6 | 4 | 2个（disc、空串） | ✅ 前 4 个正确 |
| | B-1 折扣汇总 | `reports/summary_discounts/start/end/type/loc/disc` | 7 | **#1** | `Summary_discounts` | 6 | 5 | 1个（空串） | ✅ 前 5 个正确 |
| | B-2 折扣明细 | `reports/specific_discounts/start/end/discval/type/disc` | 7 | **#17** | `Specific_discounts` | 5 | 5 | 0 | ✅ 全部正确 |
| | C-1 支付方式 | `reports/summary_payments/start/end/fake0/fakeLoc/fakeDisc` | 7 | **#1** | `Summary_payments` | 6 | 2 | 4个 | ✅ 前 2 个正确 |
| | C-2 费用分类 | `reports/summary_expenses_categories/start/end/fake0/...` | 7 | **#1** | `Summary_expenses_categories` | 6 | 3 | 3个 | ⚠️ 第3个收到假值 "0" |

---

## 六、关键发现：路由 #1 是所有 summary_* 带参数请求的总入口

**最重大的发现**：所有前缀为 `summary_` 的带参数（>=4 段）请求，无论是什么报表类型，全部命中**同一条路由 #1**：

```
reports/summary_sales/...          ↘
reports/summary_items/...          ↘
reports/summary_categories/...     ↘ 全部通过路由 #1 的动态方法名 Summary_$1
reports/summary_customers/...      ↘ 分发到对应的控制器方法
reports/summary_discounts/...      ↘
reports/summary_payments/...       ↘
reports/summary_expenses_categories/... ↘
```

路由 #1 的动态方法名机制：
```php
// $1 = summary_ 后面的部分
// 方法名 = "Summary_" . $1

// 例 1: reports/summary_sales/...
$1 = "sales" → Summary_sales()

// 例 2: reports/summary_expenses_categories/...
$1 = "expenses_categories" → Summary_expenses_categories()

// 例 3: reports/summary_discounts/...
$1 = "discounts" → Summary_discounts()
```

这就是为什么系统只需要一条通配路由就能分发到十几个不同报表方法的核心机制。

---

## 七、额外 URL 段如何继续传递参数

### 7.1 两种参数传递模式的对比

| 模式 | 触发条件 | 参数顺序来源 | 额外段处理 |
|------|---------|-------------|-----------|
| **通配路由模式**（当前系统主模式） | 命中带 `(:any)` 占位符的定义路由 | 1. 动态方法名之后的反向引用按顺序<br>2. `$multipleSegmentsOneParam=false` 时，含 `/` 的值按 `/` 再分割 | 末尾 `(:any)` 自然捕获，按 `/` 分割后多传的参数被 PHP 截断 |
| **自动路由模式**（备用模式） | 所有定义路由均不匹配时触发 | URL 段 3, 段 4, 段 5, ... 的顺序直接绑定到形参 | 多余参数被 PHP 截断 |

### 7.2 通配路由模式下的参数流转细节

以路由 #1 为例，URL 段如何变成方法实参的完整链路：

```
URL 段 1: reports
URL 段 2: summary_sales        ── 匹配单元 2 summary_(:any) → $1 = "sales"
URL 段 3: 2024-01-01           ── 匹配单元 3 (:any) → $2 = "2024-01-01"
URL 段 4: 2024-12-31           ──┐
URL 段 5: complete                ├── 匹配单元 4（末尾 (:any)）→ $3 = "2024-12-31/complete/all/0"
URL 段 6: all                     │                                  （含 /）
URL 段 7: 0                     ──┘
                                            ↓
                  路由目标替换: Reports::Summary_$1/$2/$3/$4
                                            ↓
                             Reports::Summary_sales/2024-01-01/2024-12-31/complete/all/0/
                                            ↓
                          方法名后的参数部分按 / 分割（$multipleSegmentsOneParam=false）
                                            ↓
                    ["2024-01-01", "2024-12-31", "complete", "all", "0", ""]
                                            ↓
                       方法形参: summary_sales($start, $end, $type, $loc='all')
                                            ↓
                       $start_date = "2024-01-01"  ← 参数 1
                       $end_date   = "2024-12-31"  ← 参数 2
                       $sale_type  = "complete"    ← 参数 3
                       $location_id= "all"         ← 参数 4
                       参数 5 "0"、参数 6 "" 被 PHP 静默忽略
```

**注意**：因为 `$4` 悬空是空串，所以目标路径末尾永远有一个额外的 `/`，导致最后多出一个空字符串参数。

### 7.3 末尾 `(:any)` 捕获的段数与实参数的对应关系

以路由 #1 为例，末尾 `(:any)` 单元捕获 N 个段时：

| URL 总段数 | 末尾 (:any) 捕获段数 | `$3` 的值 | `$4` | 参数部分分割后实参数 |
|-----------|---------------------|----------|------|---------------------|
| 4 段 | 1 段 | `b`（无 `/`） | `""` | 3 个：`[a, b, ""]` |
| 5 段 | 2 段 | `b/c`（含 1 个 `/`） | `""` | 4 个：`[a, b, c, ""]` |
| 6 段 | 3 段 | `b/c/d`（含 2 个 `/`） | `""` | 5 个：`[a, b, c, d, ""]` |
| 7 段 | 4 段 | `b/c/d/e`（含 3 个 `/`） | `""` | 6 个：`[a, b, c, d, e, ""]` |

**规律**：实参数 = 末尾 (:any) 捕获段数 + 2（非末尾占位符数）+ 1（悬空 `$4` 的空串）
= 总段数 - 3（reports + summary_ + 中段占位符） + 3
= 总段数

当总段数 = 7 时，实参数 = 6（与上表一致）。

### 7.4 什么时候会走自动路由？

**在当前 OSPOS 的路由定义下，summary_* 系列报表的带参请求几乎不会走自动路由。** 因为：

1. 路由 #1 的末尾 `(:any)` 可以匹配任意多个段（4 段或更多）
2. 前端生成的 URL 是 7 段，满足 >= 4 段的条件
3. 路由 #1 在所有 summary_* 字面量路由之前定义，所以会先被匹配

**走自动路由的唯一可能性**：URL 段数 < 4 且 > 2，且字面量路由不匹配。例如：
```
URL: reports/summary_sales/a（3 段）

匹配：
  路由 #1: 需要 4段+ → ❌
  路由 #2-#4: 字面量不匹配 → ❌
  路由 #5: reports/summary_(:any)（末尾 (:any)，2段+） → ✅ 匹配！
  $1 = "sales/a"（含 /，因为末尾 (:any) 可以跨段）
  调用: Reports::date_input("sales", "a")
  （$1 含 /，被分割成 2 个参数传递给 date_input()，但 date_input 无形参，参数被忽略）
```

所以 3 段 URL 会命中路由 #5（`date_input`），而不是走自动路由。

**自动路由触发条件**：必须所有定义路由（包括末尾 `(:any)` 的 2 段+通配）都不匹配才会触发。在当前报表路由体系中，这种情况几乎不存在，因为末尾 `(:any)` 的通配模式覆盖了 2 段或更多的几乎所有情况。

---

## 八、`$4` 悬空问题的实际影响

| URL 段数 | `$4` 空串带来的额外参数 | 受影响方法 | 实际影响 |
|---------|------------------------|-----------|---------|
| 4 段 | 末尾空串（3→4 个参数） | `summary_sales` 等 4 形参方法 | `$location_id` 收到空串而非默认值（但前端几乎不生成 4 段 URL） |
| 5 段 | 末尾空串（4→5 个参数） | `summary_sales` 等 4 形参方法 | 无影响（参数 5 被截断）；对 5 形参的 `summary_discounts`，第 5 形参 `$discount_type` 收到空串而非默认值 `0` |
| 6 段 | 末尾空串（5→6 个参数） | 所有方法 | 无影响（多余的 1 个参数被截断） |
| 7 段 | 末尾空串（6→7 个参数...实际还是 6 个，见上面规律分析） | 所有方法 | 无影响（多余参数都被截断） |

**结论**：`$4` 悬空问题在前端生成 7 段 URL 的场景下几乎没有实质影响，只有在生成较少段数 URL 时才可能触发 bug。路由 #6（`graphical_` 系列）同样存在 `$4` 悬空问题，影响范围相同。

---

## 九、各报表类型对应的方法分发表

| 报表 URL 前缀 | 带参（>=4段）时命中路由 | 动态方法名 | 形参数 | 实参数（7段URL） | 多余参数 |
|-------------|----------------------|-----------|--------|----------------|---------|
| `reports/summary_sales/` | #1 | `Summary_sales` | 4 | 6 | 2 |
| `reports/summary_items/` | #1 | `Summary_items` | 4 | 6 | 2 |
| `reports/summary_categories/` | #1 | `Summary_categories` | 4 | 6 | 2 |
| `reports/summary_customers/` | #1 | `Summary_customers` | 4 | 6 | 2 |
| `reports/summary_suppliers/` | #1 | `Summary_suppliers` | 4 | 6 | 2 |
| `reports/summary_employees/` | #1 | `Summary_employees` | 4 | 6 | 2 |
| `reports/summary_taxes/` | #1 | `Summary_taxes` | 4 | 6 | 2 |
| `reports/summary_sales_taxes/` | #1 | `Summary_sales_taxes` | 4 | 6 | 2 |
| `reports/summary_discounts/` | #1 | `Summary_discounts` | 5 | 6 | 1 |
| `reports/summary_payments/` | #1 | `Summary_payments` | 2 | 6 | 4 |
| `reports/summary_expenses_categories/` | #1 | `Summary_expenses_categories` | 3 | 6 | 3 |
| `reports/graphical_*/` | #6 | `Graphical_*` | 5 | 6 | 1 |
| `reports/specific_discounts/` | #17 | `Specific_discounts` | 5 | 5 | 0 |
| `reports/specific_customers/` | #17 | `Specific_customers` | 4 | 5+ | 1+ |
| `reports/specific_employees/` | #17 | `Specific_employees` | 4 | 5+ | 1+ |
| `reports/specific_suppliers/` | #17 | `Specific_suppliers` | 4 | 5+ | 1+ |
| `reports/detailed_sales/` | #14 | `Detailed_sales` | 4 | 5+ | 1+ |
| `reports/detailed_receivings/` | #14 | `Detailed_receivings` | 4 | 5+ | 1+ |
| `reports/inventory_*/`（除low/summary） | #10 | `Inventory_*` | 2 | 3+ | 1+ |

---

## 十、修正建议

### 建议 1：修复 `$4` 悬空问题

```php
// 修正前（3 个占位符，4 个反向引用）
$routes->add('reports/summary_(:any)/(:any)/(:any)', 'Reports::Summary_$1/$2/$3/$4');
$routes->add('reports/graphical_(:any)/(:any)/(:any)', 'Reports::Graphical_$1/$2/$3/$4');

// 修正后（去掉悬空的 $4，末尾 (:any) 会自然包含所有需要的参数）
$routes->add('reports/summary_(:any)/(:any)/(:any)', 'Reports::Summary_$1/$2/$3');
$routes->add('reports/graphical_(:any)/(:any)/(:any)', 'Reports::Graphical_$1/$2/$3');
```

去掉 `$4` 后，末尾多出的空串参数消失，参数个数减少 1。

### 建议 2：统一参数个数与形参匹配

当前通过 PHP 截断多余实参的方式虽然能工作，但属于"碰巧正确"。建议根据各方法实际形参数定义更精确的路由：

```php
// 对只有 2 个形参的 summary_payments，定义专用路由：
$routes->add('reports/summary_payments/(:segment)/(:segment)', 'Reports::Summary_payments/$1/$2');

// 对有 3 个形参的 summary_expenses_categories：
$routes->add('reports/summary_expenses_categories/(:segment)/(:segment)', 'Reports::Summary_expenses_categories/$1/$2');
// 并去掉方法签名中的 $sale_type 参数（因为模型层不用）
```

### 建议 3：`$multipleSegmentsOneParam` 评估

考虑升级到 `$multipleSegmentsOneParam = true`，这样末尾 `(:any)` 捕获的多段值会作为单个参数，不会意外增加实参数，使路由行为更加可预测。但这是一个全局配置变更，需要评估对其他模块路由的影响。
