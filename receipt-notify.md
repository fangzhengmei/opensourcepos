# 收据邮件/短信通知业务上下文分析

## 概述

OSPOS 系统的收据通知功能目前仅支持**邮件**发送，**短信**功能是独立模块，与收据无直接关联。邮件发送是前端驱动的异步操作，发送状态不持久化到数据库。

---

## 一、邮件收据发送入口：真实路径 vs 死代码

### 重要结论

后端定义了 **2 个**发送邮件的控制器方法，但前端**实际调用的只有 1 个**（`getSendPdf`），另一个（`getSendReceipt`）是死代码，没有任何前端代码调用它。

---

### 1. 真实使用的接口：getSendPdf

**控制器方法**：[Sales::getSendPdf()](file:///d:/fz/0601-1/solo-dogfeeding/code/19-opensourcepos/app/Controllers/Sales.php#L933-L974)

**接口**：`GET /sales/sendPdf/{sale_id}/{type}`

**特点**：
- 生成 PDF 作为邮件附件
- 邮件正文使用 `invoice_email_message` 配置的模板（支持 Token 替换）
- 支持的 Token：`$INV`（发票号）、`$CO`（销售号）、`$CU`（客户名）

---

### 2. 前端 6 个真实触发点

#### 2.1 结账完成后：5 种单据页面 + 编辑弹窗

所有这些页面都遵循相同的模式：
- 页面加载时，如果 `email_receipt == true` 且有 `customer_email` → **自动调用**
- 页面上有"发送邮件"按钮 → **手动调用**

| 页面视图 | 单据类型 | type 参数 | 自动触发条件 | 手动按钮文案 |
|---|---|---|---|---|
| [receipt.php](file:///d:/fz/0601-1/solo-dogfeeding/code/19-opensourcepos/app/Views/sales/receipt.php#L24-L58) | 收据 | `receipt` | `email_receipt` 为 true | "发送收据" |
| [invoice.php](file:///d:/fz/0601-1/solo-dogfeeding/code/19-opensourcepos/app/Views/sales/invoice.php#L32-L72) | 发票 | （默认=invoice） | `email_receipt` 为 true | "发送发票" |
| [tax_invoice.php](file:///d:/fz/0601-1/solo-dogfeeding/code/19-opensourcepos/app/Views/sales/tax_invoice.php#L32-L71) | 税务发票 | （默认=invoice） | `email_receipt` 为 true | "发送发票" |
| [quote.php](file:///d:/fz/0601-1/solo-dogfeeding/code/19-opensourcepos/app/Views/sales/quote.php#L28-L67) | 报价单 | `quote` | `email_receipt` 为 true | "发送报价" |
| [work_order.php](file:///d:/fz/0601-1/solo-dogfeeding/code/19-opensourcepos/app/Views/sales/work_order.php#L33-L70) | 工单 | `work_order` | `email_receipt` 为 true | "发送工单" |

#### 2.2 销售编辑弹窗：form.php

**位置**：[form.php](file:///d:/fz/0601-1/solo-dogfeeding/code/19-opensourcepos/app/Views/sales/form.php#L156-L171)

**触发方式**：
- 没有自动发送
- 用户点击按钮 → 弹出确认框 → 确认后调用
- **需要客户有邮箱**（`$sale_info['email']` 不为空才显示按钮）

**调用接口**：`GET /sales/sendPdf/{sale_id}`（type 默认为 invoice）

**使用场景**：在销售详情/编辑弹窗中手动发送

---

### 3. 死代码：getSendReceipt（前端无调用）

**控制器方法**：[Sales::getSendReceipt()](file:///d:/fz/0601-1/solo-dogfeeding/code/19-opensourcepos/app/Controllers/Sales.php#L983-L1008)

**接口**：`GET /sales/sendReceipt/{sale_id}`

**状态**：**前端无任何调用**，属于死代码/预留接口

**与 getSendPdf 的区别**：
- 不生成 PDF 附件
- 直接把收据 HTML 作为邮件正文（使用 `receipt_email.php` 模板）
- 没有 PDF 附件，只有 HTML 正文

**为什么会存在**：
- 可能是早期版本的遗留代码
- 也可能是为未来功能预留的接口
- 代码注释写 "Used in app/Views/sales/receipt.php"，但实际 receipt.php 调用的是 `/sales/sendPdf/.../receipt`

---

### 4. 自动发送的完整触发条件

自动发送需要同时满足以下所有条件：

1. **配置允许**：`email_receipt_check_behaviour` 不为 `never`
2. **Session 标记**：`sales_email_receipt` 为 true（或 always 模式）
3. **客户有邮箱**：`customer_email` 不为空
4. **页面渲染时变量已注入**：视图中的 `$email_receipt` 和 `$customer_email` 都有值

**自动发送只发生在"结账完成"的场景**（postComplete 返回的页面），查看历史单据时不会自动发送（因为历史查看页面的 `$email_receipt` 由 `_load_sale_data` 构建，该方法内部 `clear_all()` 清空了 Session，所以 `email_receipt` 为 false）。

---

## 二、邮件发送的业务上下文

### 1. 发送开关控制

**配置项**：`email_receipt_check_behaviour`

**配置位置**：系统设置 -> 收据配置

**可选值**（在 [Sale_lib::is_email_receipt()](file:///d:/fz/0601-1/solo-dogfeeding/code/19-opensourcepos/app/Libraries/Sale_lib.php#L546-L556) 中判断）：

| 值 | 含义 | 说明 |
|---|---|---|
| `always` | 总是发送 | 每次结账都自动勾选发送邮件 |
| `never` | 从不发送 | 始终不自动发送 |
| `last` | 记住上次 | 基于 Session 记住用户上次的选择（默认） |

**Session 存储**：`sales_email_receipt`

**设置接口**：`POST /sales/setEmailReceipt` → [Sales::postSetEmailReceipt()](file:///d:/fz/0601-1/solo-dogfeeding/code/19-opensourcepos/app/Controllers/Sales.php#L381-L385)

**前端交互**：收银台页面的"邮件收据"复选框 [register.php](file:///d:/fz/0601-1/solo-dogfeeding/code/19-opensourcepos/app/Views/sales/register.php#L525-L532)

---

### 2. email_receipt 生命周期：结账后是否影响下一单

**核心结论**：默认 `last` 模式下，勾选状态**不会延续到下一单**。

#### 时序分析

```
第一单 - 收银台页面
  ① 用户勾选"邮件收据"
      → AJAX POST /sales/setEmailReceipt
      → Sale_lib::set_email_receipt()
      → Session: sales_email_receipt = 'true' / '1'

  ② 点击"完成"按钮
      → Sales::postComplete() 入口
      → L724 读取 $data['email_receipt'] = is_email_receipt()  // 值为 true，存入视图变量
      → L816/L854/L882/L900 调用 sale->save_value() 保存销售单到数据库
      → L827/L861/L888/L919 调用 sale_lib->clear_all()
            ↓
            Sale_lib::clear_all() [L1413-L1429]
              → L1420: $this->clear_email_receipt()
              → Session 删除 sales_email_receipt 键
      → 返回 receipt.php/invoice.php 视图（视图中的 $email_receipt 仍是 true）

  ③ 收据页面 JS
      → 判断 $email_receipt == true
      → 自动触发发送邮件 AJAX（正常执行）

用户点击"回到收银台" → 开始第二单
  ④ Sales::getIndex() → _reload()
      → L1248: $data['email_receipt'] = is_email_receipt()
            ↓
            is_email_receipt() [L546-L556]
              → email_receipt_check_behaviour == 'last'
              → 检查 Session sales_email_receipt
              → 键已不存在 → 返回 false
      → 收银台页面复选框为【未勾选状态】
```

#### 三种配置模式下的表现对比

| 配置值 | 下一单默认状态 | 原因 |
|---|---|---|
| `always` | 始终勾选 | 直接返回 true，不依赖 Session |
| `never` | 始终不勾选 | 直接返回 false，不依赖 Session |
| `last`（默认） | **不勾选** | `clear_all()` 清除了 Session，记忆丢失 |

#### 关键代码位置

- 清除操作：[Sale_lib::clear_all()](file:///d:/fz/0601-1/solo-dogfeeding/code/19-opensourcepos/app/Libraries/Sale_lib.php#L1413-L1429) 第 L1420 行
- 读取判断：[Sale_lib::is_email_receipt()](file:///d:/fz/0601-1/solo-dogfeeding/code/19-opensourcepos/app/Libraries/Sale_lib.php#L546-L556)

#### 注意

`last` 模式代码注释写的是 "Remember last setting, session based though"，但由于 `clear_all()` 在销售完成时无条件清除 Session，实际上它**只在同一单的多次操作之间**（如添加/删除商品）保持记忆，**不会跨单记忆**。这是实现上与注释描述的细微偏差。

---

### 3. 前置条件检查

发送邮件前必须满足以下条件：

| 条件 | 检查位置 | 失败处理 |
|---|---|---|
| 客户邮箱不为空 | 控制器方法内 | 返回错误消息 `Sales.receipt_no_email` |
| SMTP 配置有效 | [Email_lib](file:///d:/fz/0601-1/solo-dogfeeding/code/19-opensourcepos/app/Libraries/Email_lib.php) 构造函数 | 发送失败时记录日志 |

---

## 三、邮件模板数据构建

### 1. 核心数据构建方法

**方法**：[Sales::_load_sale_data()](file:///d:/fz/0601-1/solo-dogfeeding/code/19-opensourcepos/app/Controllers/Sales.php#L1082-L1182)

**输入**：`$sale_id` 销售ID

**输出**：关联数组，包含以下主要字段：

#### 基础信息
- `sale_id` / `sale_id_num` - 销售编号
- `transaction_time` / `transaction_date` - 交易时间
- `employee` - 收银员姓名
- `comments` - 备注
- `sale_status` - 销售状态（COMPLETED / SUSPENDED / CANCELED）

#### 商品与价格
- `cart` - 购物车商品列表（含名称、价格、数量、折扣等）
- `subtotal` - 小计
- `discount` - 折扣总额
- `taxes` - 税金明细
- `total` - 总计
- `amount_due` - 应付金额
- `amount_change` - 找零

#### 支付信息
- `payments` - 支付方式列表
- `payments_total` - 支付总额
- `payments_cover_total` - 支付是否覆盖总额

#### 客户信息
- `customer` - 客户名称
- `customer_email` - 客户邮箱（发送邮件的关键）
- `customer_address` / `customer_location` - 客户地址
- `customer_discount` - 客户折扣
- `customer_rewards` - 会员积分（如有）

#### 公司信息
- `company_info` - 公司地址、电话等
- `config` - 系统配置数组

#### 邮件专用字段（发送前补充）
- `barcode` - 收据条码（base64 编码的 SVG）
- `img_tag` - 公司 Logo（base64 编码的 img 标签）
- `mimetype` - Logo 的 MIME 类型

---

### 2. 邮件模板视图

**收据邮件模板**：[receipt_email.php](file:///d:/fz/0601-1/solo-dogfeeding/code/19-opensourcepos/app/Views/sales/receipt_email.php)

**模板结构**：
- 头部：公司 Logo、公司名称、地址、电话
- 基本信息：客户名、销售号、收银员
- 商品明细表格：名称、单价、数量、金额
- 价格汇总：小计、折扣、税金、总计
- 支付明细：各支付方式金额
- 底部：退货政策、条码

---

### 3. 邮件发送库

**类**：[Email_lib](file:///d:/fz/0601-1/solo-dogfeeding/code/19-opensourcepos/app/Libraries/Email_lib.php)

**核心方法**：`sendEmail($to, $subject, $message, $attachment = null)`

**配置来源**：系统配置中的 SMTP 设置（`smtp_host`, `smtp_user`, `smtp_pass`, `smtp_port` 等）

**返回值**：布尔值，true 表示发送成功，false 表示失败

---

## 四、失败后的状态影响

### 1. 数据库状态

**重要结论**：邮件发送成功与否，**不影响销售单的任何数据库状态**。

**原因**：
- `ospos_sales` 表中**没有**邮件发送状态相关字段
- 销售单在调用邮件发送之前已经保存完成，状态已设为 `COMPLETED`
- 邮件发送是独立的异步操作（前端 AJAX 调用）

**销售表字段**（[initial_schema.sql](file:///d:/fz/0601-1/solo-dogfeeding/code/19-opensourcepos/app/Database/Migrations/sqlscripts/initial_schema.sql#L501-L513)）：
- `sale_time` - 销售时间
- `customer_id` - 客户ID
- `employee_id` - 员工ID
- `comment` - 备注
- `invoice_number` - 发票号
- `sale_id` - 销售ID（主键）
- `sale_status` - 销售状态
- `quote_number` - 报价单号
- `work_order_number` - 工单号
- `sale_type` - 销售类型
- `dinner_table_id` - 餐桌号

---

### 2. 失败处理

**邮件发送失败**：
- 返回 JSON：`{ success: false, message: "收据未发送至 {邮箱}", id: sale_id }`
- 前端显示红色错误通知
- 错误详情记录到系统日志（`log_message('error', ...)`）
- 用户可手动点击按钮重试

**无邮箱地址**：
- 返回 JSON：`{ success: false, message: "没有邮箱地址" }`
- 不记录错误日志

---

### 3. 可重试性

邮件发送是**幂等操作**，可重复调用：
- 每次调用都会重新生成邮件内容并发送
- 没有发送次数限制
- 没有"已发送"标记防止重复发送

---

## 五、短信功能说明

### 现状

**短信与收据无直接关联**。短信是独立的消息模块，不能直接发送短信收据。

**短信模块**：[Messages 控制器](file:///d:/fz/0601-1/solo-dogfeeding/code/19-opensourcepos/app/Controllers/Messages.php)

**短信功能**：
- 手动输入手机号和消息内容发送
- 可从客户详情页发起短信
- 无自动发送短信收据的功能

### 短信库

**类**：[Sms_lib](file:///d:/fz/0601-1/solo-dogfeeding/code/19-opensourcepos/app/Libraries/Sms_lib.php)

**特点**：
- 目前是占位实现，实际发送逻辑需要对接第三方 API
- 配置项：`msg_uid`（用户名）、`msg_pwd`（密码）、`msg_src`（发送方）
- 代码中预留了 textmarketer.co.uk 等 API 的示例

---

## 六、关键代码路径总览

### 路径 1：邮件收据自动发送（结账后）

```
收银台页面 (register.php)
  → 用户勾选"邮件收据"复选框
    → AJAX POST /sales/setEmailReceipt
      → Sale_lib::set_email_receipt() 保存到 Session

点击"完成"按钮
  → POST /sales/complete
    → Sales::postComplete()
      → 保存销售单到数据库（状态 COMPLETED）
      → 返回 receipt.php / invoice.php 等视图

收据页面加载
  → 检测到 email_receipt = true 且有 customer_email
    → AJAX GET /sales/sendPdf/{id}/{type}
      → Sales::getSendPdf()
        → _load_sale_data()：
            - clear_all()              ← 第 1 次清空
            - 从数据库回填销售数据
        → 生成 PDF
        → Email_lib::sendEmail() 发送
        → clear_all()                ← 第 2 次清空（无论成功/失败）
          → 返回 JSON {success, message}
            → 前端显示通知
```

---

### 路径 2：邮件收据手动发送（6 个触发点）

**发送接口统一为 getSendPdf，getSendReceipt 是死代码（前端无调用）。**

#### 2.1 单据页面手动按钮（5 种单据）

```
收据/发票/税务发票/报价单/工单 页面
  → 用户点击"发送邮件"按钮
    → AJAX GET /sales/sendPdf/{sale_id}/{type}
      → Sales::getSendPdf()
        → _load_sale_data()
        → 生成 PDF
        → Email_lib::sendEmail() 发送
        → clear_all()
          → 返回 JSON
            → 前端显示通知
```

#### 2.2 销售编辑弹窗手动发送

```
销售详情/编辑弹窗 (form.php)
  → 用户点击"发送发票"按钮
    → 弹出确认框
    → 确认后 AJAX GET /sales/sendPdf/{sale_id}
      → Sales::getSendPdf()
        → _load_sale_data()
        → 生成 PDF
        → Email_lib::sendEmail() 发送
        → clear_all()
          → 返回 JSON
            → 关闭弹窗 + 前端显示通知
```

---

### 路径 3：查看历史收据/发票（有副作用）

```
销售编辑弹窗 (form.php)
  → 用户点击"POS 12345"链接
    → 新标签页打开 GET /sales/receipt/{sale_id}
      → Sales::getReceipt()
        → _load_sale_data($sale_id)
            - clear_all()              ← 第 1 次清空（清空当前编辑的销售会话）
            - copy_entire_sale() 从数据库回填历史单据
        → clear_all()                ← 第 2 次清空（渲染完毕清理）
          → 返回 HTML 收据页面

【副作用】：如果另一个标签页正在编辑新销售，购物车等数据会被清空。
```

**同理适用于查看历史发票**：`GET /sales/invoice/{sale_id}` → `Sales::getInvoice()`

---

### 路径 4：getSendReceipt（死代码，无前端调用）

```
（无前端入口）
手动访问 GET /sales/sendReceipt/{sale_id}
  → Sales::getSendReceipt()
    → _load_sale_data()
    → 渲染 receipt_email.php 模板（无 PDF，直接 HTML 正文）
    → Email_lib::sendEmail() 发送
    → clear_all()
      → 返回 JSON
```

---

## 七、clear_all() 调用次数汇总

| 操作路径 | 调用次数 | 位置 |
|---|---|---|
| 完成销售（postComplete） | 1 | 保存成功后 |
| 发送邮件（getSendPdf） | 2 | _load_sale_data 内部 + 方法末尾 |
| 发送邮件（getSendReceipt） | 2 | _load_sale_data 内部 + 方法末尾 |
| 查看历史收据（getReceipt） | 2 | _load_sale_data 内部 + 方法末尾 |
| 查看历史发票（getInvoice） | 2 | _load_sale_data 内部 + 方法末尾 |

**一次完整结账+自动发邮件累计调用 3 次 clear_all()**

---

## 八、业务依赖总结

| 依赖项 | 类型 | 说明 |
|---|---|---|
| 客户邮箱 | 必填 | 没有邮箱则不显示发送按钮，也不会自动发送 |
| SMTP 配置 | 基础设施 | 邮件实际发送依赖正确的 SMTP 配置 |
| email_receipt_check_behaviour | 系统配置 | 控制是否默认勾选邮件发送 |
| 公司 Logo | 可选 | 显示在邮件收据头部 |
| 条码生成库 | 依赖 | 邮件底部显示销售号条码 |
| PDF 生成库 (dompdf) | 依赖 | 发送 PDF 附件时需要 |
| Session | 依赖 | 存储用户的邮件收据偏好（last 模式） |

---

## 八、潜在问题与注意事项

1. **无发送状态记录**：`ospos_sales` 表没有邮件/短信发送状态字段，无法追溯某笔销售的收据是否已发送，也无法防止重复发送
2. **前端驱动**：邮件发送依赖收据页面加载时的 JS 自动触发，如果用户在页面加载前关闭浏览器则不会发送
3. **异步失败无感**：自动发送在后台 AJAX 执行，如果失败仅显示顶部通知条，用户可能没注意到
4. **短信能力缺失**：目前没有短信收据功能，短信模块是独立的通用消息模块，与收据流程无集成
5. **无重试机制**：发送失败后需要用户手动点击按钮重试，没有自动重试队列
6. **无审计日志**：邮件发送没有专门的审计日志表，仅在失败时记录通用错误日志
7. **getSendReceipt 是死代码**：后端定义了 `getSendReceipt()` 方法，但前端没有任何代码调用它。实际所有发送都走 `getSendPdf()`
8. **last 模式跨单记忆偏差**：`email_receipt_check_behaviour=last` 配置注释写"记住上次"，但 `clear_all()` 在销售完成时清空 Session，实际**不会跨单记忆**，仅在同一单多次操作间有效
9. **多标签页会话冲突（发送邮件）**：两个标签页共享 Session，一个标签页的发送邮件操作（内部 clear_all）可能清空另一个标签页正在编辑的销售内容
10. **多标签页会话冲突（查看历史单据）**：在新标签页查看历史收据/发票（`getReceipt` / `getInvoice`）会调用 `_load_sale_data()` 和 `clear_all()`，清空当前正在编辑的购物车
11. **展示与发送数据源不一致**：页面展示的是结账时的临时内存数据，邮件里的是数据库回填的数据，虽然正常情况下一致，但存在理论上的不一致窗口
12. **clear_all() 过度调用**：完成一次结账+发送邮件，会连续调用 3 次 `clear_all()`；查看一次历史收据也会调用 2 次 `clear_all()`，虽然结果正确但略显冗余
13. **查看历史单据的副作用被低估**：`getReceipt()` / `getInvoice()` 方法名看起来是"只读"操作，但实际上有修改 Session 的副作用，违反了直觉预期
