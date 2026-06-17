# OSPOS 退货退款流程解读

> 目的：把"退货退款"在代码中的真实走向讲清楚，覆盖 **收据匹配、库存回补、支付逆向、积分回收** 四条主线。
> 核心结论：OSPOS 不存在独立的"退款单"。退货是 **复用销售流程**，通过把购物车数量取负、`sale_type` 置为 `SALE_TYPE_RETURN`，再用同一套 `save_value` 落库，从而让库存、支付、积分在"负数语义"下自然逆向。

---

## 0. 整体脉络

退货复用 `Sales` 控制器的销售登记页（register）。操作链路有两条入口：

1. 切换为退货模式 → `app/Controllers/Sales.php:254-296`（`postChangeMode`）把 `sales_mode` 设为 `return`、`sale_type` 设为 `SALE_TYPE_RETURN`。
2. 退货模式下输入内容分两种：
   - **路径 A（整单回填）**：输入原收据号 → `getItemSearch` 校验合法后，`postAdd` 命中 `return_entire_sale`，把原单商品以 **负数量** 装回购物车。
   - **路径 B（无收据 / 部分退货）**：直接输入商品条码 → 不走收据校验，`postAdd` 在退货模式下先把数量取负，再走普通 `add_item`，直接形成负数量退货项。
3. 录入退款支付 → `app/Controllers/Sales.php:393-472`（`postAddPayment`，支付金额可为负，或靠 `cash_refund` 找零退款）。
4. 完成交易 → `app/Controllers/Sales.php:691-923`（`postComplete`）调 `app/Models/Sale.php:518-688`（`save_value`）一次性写库，库存/支付/积分全部在此反转。

```
postChangeMode(return)
   └─ sale_lib.set_mode('return') / set_sale_type(SALE_TYPE_RETURN)
        │
        ├─ 路径A：输入收据号 ──getItemSearch──▶ isValidReceipt──▶ 建议列表提示
        │                       └─ postAdd ──return_entire_sale──▶ 购物车(数量取负) ──▶ set_customer(原客户)
        │
        └─ 路径B：直接输商品 ──postAdd──▶ quantity = -quantity ──▶ add_item(负数量) ──▶ 购物车(局部负项)
                                                │
postAddPayment ◀─────────────────────────────────┘
     │
postComplete ──▶ Sale::save_value
        ├─ sales_items   (quantity_purchased < 0)
        ├─ item_quantity (quantity - 负数 = 回补)
        ├─ inventory     (trans_inventory = -负数 = 入库)
        ├─ sales_payments(payment_amount / cash_refund)
        ├─ giftcard / rewards 余额回冲
        └─ save_customer_rewards(earned 取负 → 扣回已赠积分)
```

---

## 1. 收据匹配：整单回填的便利入口，不是退货的必要条件

退货的起点是"凭原收据号找到原销售"，但这只是**整单回填的便利入口**，代码上**并不阻止无依据退货**。

### 1.1 模式切换

`app/Controllers/Sales.php:254-296`（`postChangeMode`）接收前端下拉的 `mode`，落到 `sale_lib`：

- `mode == 'return'` → `set_sale_type(SALE_TYPE_RETURN)`（`app/Controllers/Sales.php:267-269`）。
- 退货模式判断由 `app/Libraries/Sale_lib.php:471-474`（`is_return_mode`）提供（读 session `sales_mode == 'return'`），后续多处分支依据它。

### 1.2 收据号合法性校验（仅用于搜索建议 + 整单回填分支）

用户在搜索框输入收据号时，`app/Controllers/Sales.php:194-209`（`getItemSearch`）在退货模式下调用 `app/Models/Sale.php:414-436`（`isValidReceipt`）：

- 解析 `"POS <数字>"` 格式 → 调 `exists(sale_id)` 确认销售存在；
- 或在启用发票时，按发票号 `get_sale_by_invoice_number` 反查，并把入参改写为 `"POS " . sale_id`（统一回 POS 编号）。

**注意**：这里只是把合法收据放进搜索建议列表，方便用户选中走整单回填；它**不构成任何拦截**——用户完全可以直接输入普通商品条码走退货（见 §1.4）。

### 1.3 整单回填（路径 A）

`app/Controllers/Sales.php:503-575`（`postAdd`）拿到输入项后：

1. **先在退货模式下无条件把数量取负**：`$quantity = ($mode == 'return') ? -$quantity : $quantity;`（`app/Controllers/Sales.php:525`）。
2. 再判断输入是否为合法收据，是则直接走 `app/Libraries/Sale_lib.php:1308-1322`（`return_entire_sale`，见 `app/Controllers/Sales.php:528-529`）。

`return_entire_sale` 是收据匹配的落点：

1. 拆 `"POS #"` 得到 `sale_id`；
2. `empty_cart()` + `remove_customer()` 清空当前登记；
3. 遍历 `app/Models/Sale.php:855`（`get_sale_items_ordered`）取原单明细，按 **`-$row->quantity_purchased`** 重新 `add_item`，价格/折扣/序列号沿用原单；
4. `set_customer(原单客户)`。

由此，购物车里出现的是原单的"镜像负数"——金额合计自然为负，构成后续逆向的基础。

### 1.4 无收据 / 部分退货（路径 B：直接输入普通商品）

这是最容易被忽略的路径，但代码上与整单回填 **完全并列**。仍然看 `app/Controllers/Sales.php:503-575`（`postAdd`）的分支结构：

```php
$quantity = ($mode == 'return') ? -$quantity : $quantity;   // L525 —— 退货模式下已经取负

if ($mode == 'return' && $this->sale->isValidReceipt(...)) {
    $this->sale_lib->return_entire_sale(...);               // 路径 A：整单回填
} elseif ($this->item_kit->is_valid_item_kit(...)) {
    ... 套件处理（数量同样为负）
} else {
    $this->sale_lib->add_item(...);                         // 路径 B：普通商品，数量为负
}
```

关键点：

- `$quantity` 的取负发生在 `if/else` 分支 **之前**（L525），所以 **无论是整单回填还是单品，数量一定是负的**。
- `else` 分支里调用的是 `add_item`——和正常销售完全相同的函数，只是传进去的 `quantity` 为负。`add_item` 本身不校验是否在退货模式，也不校验该商品是否出自某张原收据。
- 这意味着：**只要切换到退货模式，扫码任意在售商品，都能形成一笔负数量退货并正常完成**，不需要任何原始收据做支撑。合法收据校验只是为了把原单整单"镜像"回购物车，节省手工录入，并不是退货权限的闸门。

---

## 2. 库存回补：负数量如何把货加回去

库存的回补发生在 `app/Models/Sale.php:518-688`（`save_value`）的明细循环里。整段在一个数据库事务内（`transStart`/`transComplete`）。

关键条件（`app/Models/Sale.php:635`）：

```php
if ($cur_item_info->stock_type == HAS_STOCK && $sale_status == COMPLETED) { ... }
```

> 退货时 `sale_status` 仍为 `COMPLETED`（见 `app/Controllers/Sales.php:893-900`（`postComplete`）的 `else` 分支），因此库存逻辑会执行。

回补的"负负得正"机制：

- **库存余量**：读取当前余量后写回 `quantity - $item_data['quantity']`（`app/Models/Sale.php:639-647`）。退货时 `quantity` 为负，`减负数 = 加回库存`。
- **商品复活**：若 `quantity < 0` 触发 `$item->undelete()`（`app/Models/Sale.php:650-652`），把退货时已软删的商品恢复。
- **库存流水**：写入 `inventory` 表，`trans_inventory = -$item_data['quantity']`（`app/Models/Sale.php:656-665`）。退货数量为负，`-负 = 正`，表现为一笔 **正向入库**，备注为 `"POS " . $sale_id`，可追溯对应退货单。

`sales_items` 表里同样保存 `quantity_purchased` 为负（`app/Models/Sale.php:617-633`），是报表层"退货"统计的数据源。

---

## 3. 支付逆向：退款如何落到支付记录

退货总额为负（数量取负 → `get_extended_amount` 的 `bcmul(quantity, price)` 为负，见 `app/Libraries/Sale_lib.php:1609-1614`（`get_extended_amount`））。退款有两条路径：**现金找零退款（cash_refund）** 与 **负额支付**。

### 3.1 总额与找零方向

`app/Libraries/Sale_lib.php:694-783`（`get_totals`）计算出负的 `total`、负的 `amount_due`。退货时"支付是否覆盖总额"的判定是 **反向** 的（`app/Libraries/Sale_lib.php:766-770`）：

```php
if ($this->get_mode() == 'return') {
    $totals['payments_cover_total'] = $current_due > -$threshold; // 从负侧逼近 0
}
```

### 3.2 cash_refund：现金退款主通道

在 `app/Controllers/Sales.php:691-923`（`postComplete`，退货走最后的 `else` 分支 `app/Controllers/Sales.php:891-921`）：

- `amount_change = amount_due * -1`（`app/Controllers/Sales.php:769`）。`amount_due` 为负 → `amount_change` 为正，即"应退给客户的现金"。
- 若 `amount_change > 0`：有现金支付则把 `cash_refund` 挂到现金支付项上；否则新建一条 `payment_amount=0, cash_refund=amount_change` 的现金支付（`app/Controllers/Sales.php:771-787`）。
- 同时 `app/Controllers/Sales.php:758-761` 仅对非退货拦截"负总额"，给退货的合法负总额放行。

### 3.3 支付落库

`save_value` 遍历 payments 写入 `sales_payments`（`app/Models/Sale.php:579-604`），字段含 `payment_amount`、`cash_refund`、`cash_adjustment`。退款总额口径：

```php
$total_amount = payment_amount - cash_refund;   // 退款时呈负
```

礼品卡/积分支付会被特殊处理（见第 4 节）。

### 3.4 负额支付（礼品卡/积分/非现金退款）

`app/Controllers/Sales.php:393-472`（`postAddPayment`）中：

- **礼品卡**：`new_giftcard_value = 余额 - amount_due`（`app/Controllers/Sales.php:429`）。`amount_due` 为负 → 余额变 **增加**，相当于把原消费额退回卡里；`amount_tendered = min(amount_due, 余额)` 为负，作为负额支付入账。
- **积分**：同理 `new_reward_value = points - amount_due`（`app/Controllers/Sales.php:452`），负 `amount_due` 让积分 **加回**。

### 3.5 事后编辑退款（非现金退款拆分）

对已完成单的支付编辑走 `app/Controllers/Sales.php:1444-1529`（`postSave`）：读取每行 `refund_amount_$i` 与 `refund_type_$i`（`app/Controllers/Sales.php:1466-1491`）。若退款类型非现金且 `cash_refund>0`，则把它改写成一条 **新的负额支付**（`payment_amount -= cash_refund`，`cash_refund=0`，`app/Controllers/Sales.php:1477-1482`），即"非现金退款"以负额支付行表达。最终由 `app/Models/Sale.php:452`（`update`）落库，`balance_due = amount_due - amount_tendered + cash_refund`（`app/Controllers/Sales.php:1346`）。

---

## 4. 积分回收：已赠积分如何扣回

积分在退货中分两路回收，均在 `save_value` 的事务内。

### 4.1 退还"用作支付的积分"

当存在 `Sales.rewards` 支付项时（`app/Models/Sale.php:585-588`）：

```php
$cur_rewards_value = customer.points;
customer->update_reward_points_value(customer_id, $cur_rewards_value - $payment['payment_amount']);
$total_amount_used += $payment['payment_amount'];   // 退货为负
```

退货支付金额为负 → `points - 负 = points + |额|`，把原单用作支付的积分 **加回** 客户账户（落库见 `app/Models/Customer.php:239-244` `update_reward_points_value`）。

### 4.2 扣回"销售赠送的积分"

`app/Models/Sale.php:1377-1403`（`save_customer_rewards`，在 `save_value:606` 调用）：

```php
$total_amount_earned = $total_amount * $points_percent / 100;  // total_amount 为负 → earned 为负
$points = $points + $total_amount_earned;                      // 累加负值 → 扣减
customer->update_reward_points_value($customer_id, $points);
rewards->save_value(['sale_id'=>.., 'earned'=>$total_amount_earned, 'used'=>$total_amount_used]);
```

- `$total_amount` 是上一节算出的"退款总额"（负）→ `earned` 为负 → 客户积分 **减少**，回收原单按比例赠送的积分。
- 同时向 `sales_reward_points` 写入一条记录（`app/Models/Rewards.php:26-42` `save_value`），`earned` 为负、`used` 为负，完整留痕。

> 两路合起来：原单"用积分抵扣"的部分被退回账户，原单"按消费额获赠"的部分被反向扣除，闭环一致。

---

## 5. 涉及的数据表

| 表 | 退货写入含义 |
|---|---|
| `sales` | 新增一行，`sale_type = SALE_TYPE_RETURN`、`sale_status = COMPLETED` |
| `sales_items` | `quantity_purchased < 0`，金额为负 |
| `sales_payments` | `payment_amount` 可为负；`cash_refund` 记录现金退款额 |
| `item_quantities` | `quantity - (负) = 回补`（库存加回） |
| `inventory` | `trans_inventory = -负 = 正`（正向入库流水），备注 `POS <sale_id>` |
| `giftcards` | 余额 += 退款额（通过 `update_giftcard_value`） |
| `customers.points` | 退还已用积分 + 扣减已赠积分 |
| `sales_reward_points` | `earned`、`used` 均为负 |

---

## 6. 关键代码索引

- 模式切换：`app/Controllers/Sales.php:254-296`（`postChangeMode`）
- 模式判断：`app/Libraries/Sale_lib.php:471-474`（`is_return_mode`）、`app/Libraries/Sale_lib.php:862`（`get_mode`）
- 收据搜索建议：`app/Controllers/Sales.php:194-209`（`getItemSearch`）
- 收据校验：`app/Models/Sale.php:414-436`（`isValidReceipt`）
- 添加商品（含两条退货入口）：`app/Controllers/Sales.php:503-575`（`postAdd`）
- 整单回填：`app/Libraries/Sale_lib.php:1308-1322`（`return_entire_sale`）
- 原单明细查询：`app/Models/Sale.php:855`（`get_sale_items_ordered`）
- 加商品到购物车（负数量也走这里）：`app/Libraries/Sale_lib.php:1034`（`add_item`）
- 完成/落库：`app/Controllers/Sales.php:691-923`（`postComplete`）、`app/Models/Sale.php:518-688`（`save_value`）
- 支付录入：`app/Controllers/Sales.php:393-472`（`postAddPayment`）
- 支付编辑/事后退款：`app/Controllers/Sales.php:1444-1529`（`postSave`）、`app/Models/Sale.php:452`（`update`）
- 加入支付记录：`app/Libraries/Sale_lib.php:585-612`（`add_payment`）
- 支付合计：`app/Libraries/Sale_lib.php:669`（`get_payments_total`）
- 总额/找零：`app/Libraries/Sale_lib.php:694-783`（`get_totals`）
- 明细金额计算（承载负数量）：`app/Libraries/Sale_lib.php:1609-1614`（`get_extended_amount`）
- 积分回收：`app/Models/Sale.php:1377-1403`（`save_customer_rewards`）
- 积分流水落库：`app/Models/Rewards.php:26-42`（`save_value`）
- 客户积分更新：`app/Models/Customer.php:239-244`（`update_reward_points_value`）
