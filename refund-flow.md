# OSPOS 退货退款流程解读

> 目的：把"退货退款"在代码中的真实走向讲清楚，覆盖 **收据匹配、库存回补、支付逆向、积分回收** 四条主线。
> 核心结论：OSPOS 不存在独立的"退款单"。退货是 **复用销售流程**，通过把购物车数量取负、`sale_type` 置为 `SALE_TYPE_RETURN`，再用同一套 `save_value` 落库，从而让库存、支付、积分在"负数语义"下自然逆向。

---

## 0. 整体脉络

退货复用 `Sales` 控制器的销售登记页（register）。操作链路有三条入口：

1. 切换为退货模式 → `app/Controllers/Sales.php:254-296`（`postChangeMode`）把 `sales_mode` 设为 `return`、`sale_type` 设为 `SALE_TYPE_RETURN`。
2. 退货模式下输入内容分三种：
   - **路径 A（整单回填）**：输入原收据号 → `getItemSearch` 校验合法后，`postAdd` 命中 `return_entire_sale`，把原单商品以 **负数量** 装回购物车。
   - **路径 B（无收据 / 部分退货）**：直接输入商品条码 → 不走收据校验，`postAdd` 在退货模式下先把数量取负，再走普通 `add_item`，直接形成负数量退货项。
   - **路径 C（套件退货）**：输入套件编号（KIT #）→ 主商品（kit_item）数量取负，组成项数量在 `add_item_kit` 内部读取套件配置时为正，出现符号分裂。
3. 录入退款支付 → `app/Controllers/Sales.php:393-472`（`postAddPayment`，支付金额可为负，或靠 `cash_refund` 找零退款）。
4. 完成交易 → `app/Controllers/Sales.php:691-923`（`postComplete`）调 `app/Models/Sale.php:518-688`（`save_value`）一次性写库，库存/支付/积分全部在此反转。

```
postChangeMode(return)
   └─ sale_lib.set_mode('return') / set_sale_type(SALE_TYPE_RETURN)
        │
        ├─ 路径A：输入收据号 ──getItemSearch──▶ isValidReceipt──▶ 建议列表提示
        │                       └─ postAdd ──return_entire_sale──▶ 购物车(数量取负) ──▶ set_customer(原客户)
        │
        ├─ 路径B：直接输商品 ──postAdd──▶ quantity = -quantity ──▶ add_item(负数量) ──▶ 购物车(局部负项)
        │
        └─ 路径C：输套件编号 ──postAdd──┬─ 主商品 kit_item: quantity 已取负 ──▶ add_item(负数量)
                                      └─ 组成项: add_item_kit 内部 quantity>0 ──▶ add_item(正数量)
                                                │
postAddPayment ◀─────────────────────────────────┘
     │
postComplete ──▶ Sale::save_value
        ├─ sales_items   (kit_item 负, component 正)
        ├─ item_quantity (仅 HAS_STOCK 项: 负数量回补, 正数量扣减 → 相互抵消)
        ├─ inventory     (同上, 流水也一正一负抵消)
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
    ... 套件处理（主商品 quantity 已取负，组成项 quantity 不变仍为正）
} else {
    $this->sale_lib->add_item(...);                         // 路径 B：普通商品，数量为负
}
```

关键点：

- `$quantity` 的取负发生在 `if/else` 分支 **之前**（L525），所以 **无论是整单回填还是单品，数量一定是负的**。
- `else` 分支里调用的是 `add_item`——和正常销售完全相同的函数，只是传进去的 `quantity` 为负。`add_item` 本身不校验是否在退货模式，也不校验该商品是否出自某张原收据。
- 这意味着：**只要切换到退货模式，扫码任意在售商品，都能形成一笔负数量退货并正常完成**，不需要任何原始收据做支撑。合法收据校验只是为了把原单整单"镜像"回购物车，节省手工录入，并不是退货权限的闸门。

### 1.5 套件退货（路径 C：主商品负、组成项正的符号分裂）

套件退货是三条路径里**唯一不一定全为负数量**的情况，也是符号分裂的根源。代码位于 `app/Controllers/Sales.php:530-565`（`postAdd` 的 `elseif` 分支），处理分两步：

**第一步：添加主商品（kit_item）→ 数量为负**

`app/Controllers/Sales.php:551-557` 先处理 `kit_item_id`（套件自身的代表商品），传入的 `$quantity` 已经在 L525 被取负，因此主商品购物车项 `quantity < 0`：

```php
if (!empty($kit_item_id)) {
    if (!$this->sale_lib->add_item($kit_item_id, $item_location, $quantity, $discount, $discount_type, PRICE_MODE_KIT, ...)) {
```

主商品的 `price_mode = PRICE_MODE_KIT`，价格是否为 0 取决于 `kit_price_option`（`app/Libraries/Sale_lib.php:1055-1062`）。如果配置为"套件统一定价"，主商品价格为套件总价、组成项价格归零；如果配置为"分项计价"，则主商品价格归零、组成项各有价格。无论哪种定价模式，**主商品的 `quantity` 符号由 L525 决定，为负**。

**第二步：添加组成项（components）→ 数量为正**

`app/Controllers/Sales.php:561` 调用 `add_item_kit`，**没有把取负后的 `$quantity` 传进去**——这是符号分裂的关键：

```php
if (!$this->sale_lib->add_item_kit($item_id_or_number_or_item_kit_or_receipt, $item_location, $discount, $discount_type, $kit_price_option, $kit_print_option, $stock_warning)) {
```

在 `app/Libraries/Sale_lib.php:1334-1351`（`add_item_kit`）内部，遍历套件定义读取的 `$item_kit_item['quantity']` 是**套件配置中的原始正数**（例如"1 个汉堡套件 = 2 个面包 + 1 个肉饼"，这里的 2 和 1 都是正数）。它没有被外层的 L525 取负逻辑影响，直接以正数量传给 `add_item`：

```php
foreach ($this->item_kit_items->get_info($item_kit_id) as $item_kit_item) {
    $result &= $this->add_item($item_kit_item['item_id'], $item_location, $item_kit_item['quantity'], ...);
    //                                                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //                                                     套件配置中的原始正数，未被取负
}
```

最终购物车里出现**符号分裂**的两条（或更多）记录：

| 角色 | 商品 | quantity 符号 | 来源 |
|---|---|---|---|
| 主商品 | kit_item | **负** | 外层 L525 `$quantity = -$quantity` |
| 组成项 1 | component A | **正** | `add_item_kit` 内部读取套件配置的原始正数 |
| 组成项 2 | component B | **正** | 同上 |

> 注意：整单回填（路径 A）如果原单含套件，走的是 `return_entire_sale` 把原单 `sales_items` 逐行 `-$row->quantity_purchased` 重新 `add_item`，**不会触发路径 C 的符号分裂**。只有手动输入 `KIT #` 走套件退货时才会出现此现象。

---

## 2. 库存回补：负数量如何把货加回去（含套件的符号分裂处理）

库存的回补发生在 `app/Models/Sale.php:518-688`（`save_value`）的明细循环里。整段在一个数据库事务内（`transStart`/`transComplete`）。

关键条件（`app/Models/Sale.php:635`）：

```php
if ($cur_item_info->stock_type == HAS_STOCK && $sale_status == COMPLETED) { ... }
```

> 退货时 `sale_status` 仍为 `COMPLETED`（见 `app/Controllers/Sales.php:893-900`（`postComplete`）的 `else` 分支），因此库存逻辑会执行。

### 2.1 普通退货（路径 A/B）：负负得正

- **库存余量**：读取当前余量后写回 `quantity - $item_data['quantity']`（`app/Models/Sale.php:639-647`）。退货时 `quantity` 为负，`减负数 = 加回库存`。
- **商品复活**：若 `quantity < 0` 触发 `$item->undelete()`（`app/Models/Sale.php:650-652`），把退货时已软删的商品恢复。
- **库存流水**：写入 `inventory` 表，`trans_inventory = -$item_data['quantity']`（`app/Models/Sale.php:656-665`）。退货数量为负，`-负 = 正`，表现为一笔 **正向入库**，备注为 `"POS " . $sale_id`，可追溯对应退货单。

`sales_items` 表里同样保存 `quantity_purchased` 为负（`app/Models/Sale.php:617-633`），是报表层"退货"统计的数据源。

### 2.2 套件退货（路径 C）：主商品与组成项相互抵消

套件退货的符号分裂（主商品负、组成项正）在库存层有特殊表现。关键在于主商品（kit_item）和组成项（components）的 **`stock_type` 配置不同**，导致是否进入库存逻辑分支、以及进入后的符号方向都不同。

#### 情形 1：主商品 stock_type != HAS_STOCK（最常见）

主商品通常是"非库存商品"（`stock_type = NON_STOCK`，即套件只是虚拟组合，不独立持有库存）。此时 `app/Models/Sale.php:635` 的条件不成立，主商品 **跳过库存处理**，不会产生任何库存变动。

组成项通常是"库存商品"（`stock_type = HAS_STOCK`），它们的 `quantity` 为正 → 进入库存分支后：

- 余量：`quantity - 正数 = 减少库存`（相当于正常扣减）
- 流水：`trans_inventory = -正数 = 负`（出库流水）

**结果**：主商品是虚拟项无库存变化，组成项却在做**正向扣减**——这与"退货应该回补库存"的直觉相反。实际中只有**整单回填（路径 A）退货套件**时正确，因为原单明细里组成项也为负，重新 `-$row->quantity_purchased` 后组成项为正，但主商品可能根本不在原单里（取决于套件打印/保存配置），所以不会产生符号分裂。

> 这是一个值得注意的设计特征：手动 `KIT #` 套件退货（路径 C）在库存层实际上可能把组成项的库存**扣减**了，而不是回补。除非 kit_item 本身配置为 HAS_STOCK（见下）。

#### 情形 2：主商品 stock_type == HAS_STOCK（较少见）

如果 kit_item 本身是持有库存的真实商品（`HAS_STOCK`），则主商品 `quantity < 0` 会走正常的"负负得正"回补逻辑：

- 主商品：`quantity - 负数 = 回补库存`（加回）
- 组成项：`quantity - 正数 = 扣减库存`（减去）

如果套件定义是"1 个 kit_item = 2 个 compA + 1 个 compB"，则 kit_item 回补 1 件，compA 扣减 2 件，compB 扣减 1 件——这相当于"拆套件回收入库"，即把组合好的套件拆回散件。是否符合业务预期取决于套件是否预先组装。

#### 情形 3：组成项为非库存商品（stock_type != HAS_STOCK）

如果某个组成项是非库存商品（例如服务费、虚拟商品），它同样跳过 `HAS_STOCK` 分支，不产生库存变动，仅在 `sales_items` 留痕。

### 2.3 商品复活的不对称性

`app/Models/Sale.php:650-652` 的 `undelete` 逻辑只判断 `quantity < 0`。在套件退货中：

- 主商品 `quantity < 0` → 可能触发 `undelete`（如果已被软删）
- 组成项 `quantity > 0` → 不会触发 `undelete`

如果主商品被软删过，退货时会被复活；组成项则不会——这也是符号分裂带来的不对称行为。

---

## 3. 支付逆向：退款如何落到支付记录（含套件金额的符号来源）

退货总额为负（数量取负 → `get_extended_amount` 的 `bcmul(quantity, price)` 为负，见 `app/Libraries/Sale_lib.php:1609-1614`（`get_extended_amount`））。退款有两条路径：**现金找零退款（cash_refund）** 与 **负额支付**。

### 3.1 总额与找零方向（含套件对金额符号的影响）

`app/Libraries/Sale_lib.php:694-783`（`get_totals`）遍历购物车累加金额，不区分 item_type，只按 `quantity × price` 计算。这意味着：

- **普通退货（A/B）**：所有项 `quantity < 0`，累加后 `total < 0`、`amount_due < 0`。
- **套件退货（C）**：金额符号取决于谁在"承担金额"。金额由 `PRICE_MODE_KIT` 下的 `kit_price_option` 决定（`app/Libraries/Sale_lib.php:1055-1062`）：

| kit_price_option 配置 | 谁承担金额 | 主商品 quantity | 主商品 price | 组成项 quantity | 组成项 price | 金额合计 |
|---|---|---|---|---|---|---|
| PRICE_OPTION_KIT（套件统一定价） | 主商品 | 负（L525） | = 套件总价 | 正（配置） | = 0 | **负**（主商品负 × 正价） |
| PRICE_OPTION_ALL（分项计价） | 组成项 | 负（L525） | = 0 | 正（配置） | = 分项单价 | **正**（组成项正 × 正价） |
| PRICE_OPTION_KIT_STOCK | 库存组成项 | 负（L525） | 库存项有价格 | 正（配置） | 非库存项 0 | 取决于库存组成项价 × 正数量 |

**关键结论**：套件退货（路径 C）**不一定产生负总额**。当配置为 `PRICE_OPTION_ALL`（分项计价）时，组成项 `quantity > 0` × `price > 0` → 金额为正，主商品价格归零 → 购物车合计为**正**，这会让 `amount_due > 0`，表现为"应该向客户收钱"而非退款，与退货语义相反。

退货时"支付是否覆盖总额"的判定在 `app/Libraries/Sale_lib.php:766-770`，但这条判定只看 `mode == 'return'`，**不关心总额实际符号**：

```php
if ($this->get_mode() == 'return') {
    $totals['payments_cover_total'] = $current_due > -$threshold; // 从负侧逼近 0
}
```

如果套件退货（路径 C）产生了**正总额**，`current_due = total - payment_total` 为正，那么 `current_due > -$threshold` 几乎总是成立（正数一定大于负数阈值），所以 `payments_cover_total` 会被误判为 `true`，允许完成交易——但此时 `amount_due > 0`，意味着系统认为客户**还需付钱**，逻辑上与退货相悖。

**修正提示**：`postComplete` 的 L758-L761 只拦截"非退货模式下的负总额"，对退货模式的正总额没有对称拦截。如果业务上要防止套件退货产生正向收费，需要在这里加一条对称判断。

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

## 4. 积分回收：已赠积分如何扣回（含套件符号分裂的影响）

积分在退货中分两路回收，均在 `save_value` 的事务内。计算依据是 `payment_amount`（用于"用作支付的积分"）和 `total_amount`（用于"销售赠送的积分"），两者都**不受 item 级符号分裂直接影响**，但最终符号由购物车总额决定。

### 4.1 退还"用作支付的积分"

当存在 `Sales.rewards` 支付项时（`app/Models/Sale.php:585-588`）：

```php
$cur_rewards_value = customer.points;
customer->update_reward_points_value(customer_id, $cur_rewards_value - $payment['payment_amount']);
$total_amount_used += $payment['payment_amount'];   // 退货为负
```

退货支付金额为负 → `points - 负 = points + |额|`，把原单用作支付的积分 **加回** 客户账户（落库见 `app/Models/Customer.php:239-244` `update_reward_points_value`）。

> 套件退货（路径 C）不影响这一路，因为它只看支付记录，支付记录的符号是客户在 `postAddPayment` 录入时决定的。

### 4.2 扣回"销售赠送的积分"

`app/Models/Sale.php:1377-1403`（`save_customer_rewards`，在 `save_value:606` 调用）：

```php
$total_amount_earned = $total_amount * $points_percent / 100;  // total_amount 符号决定 earned 符号
$points = $points + $total_amount_earned;
customer->update_reward_points_value($customer_id, $points);
rewards->save_value(['sale_id'=>.., 'earned'=>$total_amount_earned, 'used'=>$total_amount_used]);
```

这里的 `$total_amount` 是上一节 `payment_amount - cash_refund` 的汇总。它的符号在套件退货（路径 C）场景下取决于购物车总额的符号：

- 普通退货（A/B）：`total < 0` → `earned < 0` → 客户积分减少（正常扣回赠送积分）。
- 套件退货（C）- PRICE_OPTION_KIT：`total < 0` → 同上正常扣回。
- 套件退货（C）- PRICE_OPTION_ALL：`total > 0` → `earned > 0` → 客户积分**反而增加**，等于"退了套件还送积分"——与退货语义完全相反。

同时向 `sales_reward_points` 写入一条记录（`app/Models/Rewards.php:26-42` `save_value`），`earned` 和 `used` 的符号跟随 `total_amount` 和 `payment_amount`。

> 两路合起来：原单"用积分抵扣"的部分被退回账户，原单"按消费额获赠"的部分被反向扣除——**但在套件退货且分项计价时，第二路会反向增加积分**，是符号分裂引发的另一个不一致点。

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
- 添加商品（含三条退货入口）：`app/Controllers/Sales.php:503-575`（`postAdd`）
- 整单回填：`app/Libraries/Sale_lib.php:1308-1322`（`return_entire_sale`）
- 原单明细查询：`app/Models/Sale.php:855`（`get_sale_items_ordered`）
- 加商品到购物车（负数量也走这里）：`app/Libraries/Sale_lib.php:1034`（`add_item`）
- 套件组成项添加（数量未取负的关键）：`app/Libraries/Sale_lib.php:1334-1351`（`add_item_kit`）
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
