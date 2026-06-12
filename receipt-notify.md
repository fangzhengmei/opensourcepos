# 收据邮件/短信通知业务上下文分析

## 概述

OSPOS 系统的收据通知功能目前仅支持**邮件**发送，**短信**功能是独立模块，与收据无直接关联。邮件发送是前端驱动的异步操作，发送状态不持久化到数据库。

---

## 一、邮件收据发送入口

### 1. 结账后自动发送（主要路径）

**触发位置**：[receipt.php](file:///d:/fz/0601-1/solo-dogfeeding/code/19-opensourcepos/app/Views/sales/receipt.php#L24-L46)

**触发时机**：销售结账完成后，页面加载时自动执行

**触发条件**（同时满足）：
- 客户有邮箱地址（`customer_email` 不为空）
- `email_receipt` 标志为 true

**前端逻辑**：
```javascript
// 页面加载后，如果 email_receipt 为 true，则自动调用发送接口
<?php if (!empty($email_receipt)): ?>
    send_email();  // 自动调用
<?php endif; ?>
```

**调用接口**：`GET /sales/sendPdf/{sale_id}/receipt`

---

### 2. 手动点击发送

**触发位置**：收据页面上的"发送收据"按钮

**按钮显示条件**：客户有邮箱地址时才显示

**调用接口**：与自动发送相同，`GET /sales/sendPdf/{sale_id}/receipt`

---

### 3. 销售详情页发送

**控制器方法**：[Sales::getSendReceipt()](file:///d:/fz/0601-1/solo-dogfeeding/code/19-opensourcepos/app/Controllers/Sales.php#L983-L1008)

**接口**：`GET /sales/sendReceipt/{sale_id}`

**用途**：从销售管理页面手动重发收据邮件

**特点**：
- 使用 `receipt_email.php` 作为邮件正文模板（HTML 格式）
- 不带 PDF 附件
- 直接嵌入收据内容到邮件正文中

---

### 4. PDF 发票/单据发送

**控制器方法**：[Sales::getSendPdf()](file:///d:/fz/0601-1/solo-dogfeeding/code/19-opensourcepos/app/Controllers/Sales.php#L933-L974)

**接口**：`GET /sales/sendPdf/{sale_id}/{type}`

**支持的类型**：
- `invoice` - 发票（默认）
- `receipt` - 收据
- `quote` - 报价单
- `work_order` - 工单

**特点**：
- 生成 PDF 作为邮件附件
- 邮件正文使用 `invoice_email_message` 配置的模板（支持 Token 替换）
- 支持的 Token：`$INV`（发票号）、`$CO`（销售号）、`$CU`（客户名）

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

### 2. 前置条件检查

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

### 邮件收据发送（自动发送路径）

```
收银台页面 (register.php)
  → 用户勾选"邮件收据"复选框
    → AJAX POST /sales/setEmailReceipt
      → Sale_lib::set_email_receipt() 保存到 Session

点击"完成"按钮
  → POST /sales/complete
    → Sales::postComplete()
      → 保存销售单到数据库（状态 COMPLETED）
      → 返回 receipt.php 视图

收据页面加载 (receipt.php)
  → 检测到 email_receipt = true 且有 customer_email
    → AJAX GET /sales/sendPdf/{id}/receipt
      → Sales::getSendPdf()
        → _load_sale_data() 构建数据
        → 生成 PDF
        → Email_lib::sendEmail() 发送
          → 返回成功/失败
            → 前端显示通知
```

### 邮件收据手动发送路径

```
销售详情页 / 收据页面
  → 点击"邮件发送"按钮
    → AJAX GET /sales/sendReceipt/{id}
      → Sales::getSendReceipt()
        → _load_sale_data() 构建数据
        → 渲染 receipt_email.php 模板
        → Email_lib::sendEmail() 发送
          → 返回成功/失败
            → 前端显示通知
```

---

## 七、业务依赖总结

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

1. **无发送状态记录**：无法追溯某笔销售的收据是否已发送，也无法防止重复发送
2. **前端驱动**：邮件发送依赖前端页面加载，如果用户关闭页面则不会发送
3. **异步失败无感**：自动发送在后台执行，如果失败用户可能没注意到通知
4. **短信能力缺失**：目前没有短信收据功能，短信模块是独立的
5. **无重试机制**：发送失败后需要用户手动点击重试，没有自动重试
6. **无审计日志**：邮件发送没有专门的审计日志，仅记录通用错误日志
