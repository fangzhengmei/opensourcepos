# OSPOS 版本升级：迁移脚本与配置补齐配合机制

## 一、核心类与文件概览

| 职责 | 文件 | 关键类/函数 |
|------|------|-------------|
| 自定义迁移扩展 | [MY_Migration.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Libraries/MY_Migration.php) | `MY_Migration` 继承 `MigrationRunner` |
| 迁移助手函数 | [migration_helper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Helpers/migration_helper.php) | `execute_script()`, `executeScriptWithTransaction()` |
| 迁移配置 | [Migrations.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Config/Migrations.php) | `$table`, `$enabled`, `$timestampFormat` |
| OSPOS 配置加载 | [OSPOS.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Config/OSPOS.php) | `set_settings()`, `getDefaultSettings()` |
| 应用配置模型 | [Appconfig.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Models/Appconfig.php) | `save()`, `batch_save()`, `exists()` |
| 登录入口控制器 | [Login.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Controllers/Login.php) | `index()`, `migrate()` |
| 配置加载事件 | [Load_config.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Events/Load_config.php) | `load_config()` |
| 事件注册 | [Events.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Config/Events.php) | `post_controller_constructor` 事件 |

---

## 二、数据库版本追踪机制

### 2.1 版本存储表 `migrations`

**表结构**（由 CI4 `MigrationRunner::ensureTable()` 自动创建）：
```sql
CREATE TABLE `migrations` (
    `id`         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `version`    VARCHAR(255) NOT NULL,      -- 迁移时间戳版本号
    `class`      VARCHAR(255) NOT NULL,      -- 迁移类名
    `namespace`  VARCHAR(255) NOT NULL,      -- 命名空间
    `group`      VARCHAR(255) NOT NULL,      -- 数据库组
    `time`       INT NOT NULL,               -- 执行时间戳
    `batch`      INT NOT NULL                -- 批次号
)
```

**版本号格式**：`YmdHis`，例如 `20250716170000`

### 2.2 版本读取与比较

代码路径：[MY_Migration.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Libraries/MY_Migration.php)

```php
// 获取代码中最新迁移版本（从文件名解析）
public function get_latest_migration(): int
{
    $migrations = $this->findMigrations();          // 扫描所有迁移文件
    return (int) basename(end($migrations)->version); // 取最后一个的版本号
}

// 获取数据库当前版本（从 migrations 表查询）
public static function get_current_version(): int
{
    try {
        $db = Database::connect();
        if ($db->tableExists('migrations')) {
            $builder = $db->table('migrations');
            $builder->select('version')->orderBy('version', 'DESC')->limit(1);
            $result = $builder->get()->getRow();
            return $result ? (int) $result->version : 0;
        }
    } catch (\Exception $e) {
        return 0;  // 数据库不可用或未初始化
    }
    return 0;
}

// 比较判断是否已是最新版本
public function is_latest(): bool
{
    return $this->get_latest_migration() === $this->get_current_version();
}
```

**关键特性**：
- `get_current_version()` 是静态方法，可在任意位置调用
- 数据库连接失败或表不存在时返回 `0`（表示全新安装）
- 版本号以整数形式比较，天然有序

---

## 三、迁移执行完整流程

### 3.1 触发入口

**入口 1：登录页面自动检测**  
[Login.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Controllers/Login.php#L23-L57)

```php
public function index(): string|RedirectResponse
{
    $migration = new MY_Migration(config('Migrations'));
    
    // 步骤 1：CI3 → CI4 迁移表转换检查
    $migration->migrate_to_ci4();
    
    $data = [
        'is_new_install' => !(MY_Migration::get_current_version()),
        'is_latest'      => $migration->is_latest(),
        'latest_version' => $migration->get_latest_migration(),
    ];
    
    if ($this->request->getMethod() !== 'POST') {
        return view('login', $data);
    }
    
    // 步骤 2：非最新版本时自动执行迁移
    if (!$data['is_latest'] || $data['is_new_install']) {
        set_time_limit(3600);
        $migration->setNamespace('App')->latest();  // 核心执行入口
        return redirect()->to('login');
    }
    // ... 登录验证逻辑
}
```

**入口 2：AJAX 接口（全新安装使用）**  
[Login.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Controllers/Login.php#L77-L99)

```php
public function migrate(): ResponseInterface
{
    $migration = new MY_Migration(config('Migrations'));
    $migration->migrate_to_ci4();
    set_time_limit(3600);
    $migration->setNamespace('App')->latest();
    return $this->response->setJSON(['success' => true]);
}
```

**前端触发逻辑**：[login.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Views/login.php#L257-L287)

### 3.2 CI3 → CI4 迁移表转换

代码路径：[MY_Migration.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Libraries/MY_Migration.php#L58-L110)

**CI3 与 CI4 迁移表区别**：
- CI3：只有 `version` 字段，无 `id` 主键
- CI4：有 `id`, `version`, `class`, `namespace`, `group`, `time`, `batch` 字段

```php
public function migrate_to_ci4(): void
{
    $ci3_migrations_version = $this->ci3_migrations_exists();
    if ($ci3_migrations_version) {
        $this->migrate_table($ci3_migrations_version);
    }
}

// 检测 CI3 表：表存在但无 id 字段
private function ci3_migrations_exists(): bool|string
{
    if ($this->db->tableExists('migrations') 
        && !$this->db->fieldExists('id', 'migrations')) {
        // 返回 CI3 中记录的版本号
    }
    return false;
}

// 转换流程
private function migrate_table(string $ci3_migrations_version): void
{
    $this->convert_table();  // 删旧表 → 建 CI4 新表
    
    // 将 CI3 已执行版本对应的迁移记录批量插入
    $available_migrations = $this->get_available_migrations();
    foreach ($available_migrations as $version => $path) {
        if ($version > (int)$ci3_migrations_version) {
            break;  // 只标记 CI3 时代已执行的迁移
        }
        $migration = new stdClass();
        $migration->version = $version;
        $migration->class = $path;
        $migration->namespace = 'App';
        $this->addHistory($migration, 1);  // 全部标记为 batch=1
    }
}
```

### 3.3 CI4 `latest()` 核心执行流程

基于 CI4 4.7.2 `MigrationRunner::latest()` 源码逻辑：

```php
public function latest(?string $group = null)
{
    $this->ensureTable();              // 确保 migrations 表存在
    
    $migrations = $this->findMigrations();  // 扫描文件 → 按版本升序排序
    
    // 过滤：移除已在历史记录中的迁移
    foreach ($this->getHistory((string) $group) as $history) {
        unset($migrations[$this->getObjectUid($history)]);
    }
    
    $batch = $this->getLastBatch() + 1;  // 新批次号
    
    foreach ($migrations as $migration) {
        if ($this->migrate('up', $migration)) {  // 执行单个迁移
            $this->addHistory($migration, $batch);  // 成功 → 写入历史
        } else {
            $this->regress(-1);  // 失败 → 回滚当前批次
            throw new RuntimeException('Migration failed');
        }
    }
    return true;
}
```

**关键方法解析**：

| 方法 | 作用 |
|------|------|
| `ensureTable()` | 检查表是否存在，不存在则创建 |
| `findMigrations()` | 扫描 `app/Database/Migrations/` 下所有 PHP 文件，按文件名时间戳升序排列 |
| `getHistory()` | 查询 `migrations` 表中已执行记录 |
| `getObjectUid()` | 生成迁移唯一标识 = 版本号 + 类名 |
| `getLastBatch()` | 获取当前最大批次号 |
| `addHistory()` | 迁移成功后写入 `migrations` 表 |
| `regress(-1)` | 回滚上一个批次中所有已执行的迁移 |

### 3.4 单个迁移的执行 `migrate('up', $migration)`

以 [20250716170000_MissingConfigKeys.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/20250716170000_MissingConfigKeys.php) 为例：

```php
class Migration_MissingConfigKeys extends Migration
{
    public function up(): void
    {
        helper('migration');
        executeScriptWithTransaction(APPPATH . 'Database/Migrations/sqlscripts/3.4.2_missing_config_keys.sql');
    }
}
```

**两种 SQL 执行方式**（[migration_helper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Helpers/migration_helper.php)）：

1. **非事务执行** `execute_script()`
   - 逐条执行 SQL 语句
   - 某条失败记录日志但继续执行后续语句
   - 返回整体是否全部成功
   - 适用于包含 DDL 语句的脚本（MySQL 中 DDL 无法回滚）

2. **事务包裹执行** `executeScriptWithTransaction()`
   - `$db->transStart()` 开启事务
   - 任一条 SQL 失败 → `transRollback()` 全部回滚
   - 全部成功 → `transComplete()` 提交
   - 适用于纯数据操作的脚本（如配置补齐）

---

## 四、中断恢复机制

### 4.1 批次（Batch）管理

**批次设计**：
- 每次调用 `latest()` 生成一个递增的 batch 号
- 同一批次内的迁移作为一个逻辑单元
- `migrations` 表的 `batch` 字段记录属于哪个批次

**失败回滚流程**：
```
批次 N 开始
  ├─ 迁移 A 执行成功 → addHistory(batch=N) ✓
  ├─ 迁移 B 执行成功 → addHistory(batch=N) ✓
  └─ 迁移 C 执行失败 ✗
      ├─ 触发 regress(-1)
      ├─ 查询 batch=N 的所有历史记录
      ├─ 按倒序调用每个迁移的 down() 方法
      ├─ 逐条 removeHistory() 删除记录
      └─ 抛出异常终止
```

**代码位置**：`MigrationRunner::regress()` 方法核心逻辑：
```php
public function regress(int $targetBatch = 0)
{
    $batches = $this->getBatches();
    
    // 倒序取出需要回滚的批次
    while ($batch = array_pop($batches)) {
        if ($batch <= $targetBatch) break;
        
        // 按倒序取出该批次的所有迁移
        foreach ($this->getBatchHistory($batch, 'desc') as $history) {
            $migration = $allMigrations[$this->getObjectUid($history)];
            if ($this->migrate('down', $migration)) {
                $this->removeHistory($history);  // 回滚成功 → 删除历史
            }
        }
    }
}
```

### 4.2 幂等性与重试

**自动重试机制**：
1. 迁移失败后，当前批次的历史记录被全部清除
2. 用户刷新登录页面 → 重新执行 `index()`
3. `is_latest()` 检测到仍非最新 → 再次调用 `latest()`
4. 重新计算待执行迁移列表，从上次失败的位置继续

**幂等性保障手段**：

| 手段 | 代码示例 | 适用场景 |
|------|----------|----------|
| `INSERT IGNORE` | `$this->db->table('app_config')->ignore(true)->insertBatch(...)` | 配置补齐、数据插入 |
| 存在性检查 | `if (!indexExists($table, $index)) { ... }` | 索引、外键、列的添加 |
| 先删后插 | `dropForeignKeyConstraints()` → 执行 → `recreateForeignKeyConstraints()` | 表结构变更（如字符集转换） |
| 跳过已执行 | CI4 自动对比 `migrations` 历史 | 所有迁移文件 |

### 4.3 特殊场景：CI4 转换中断

如果在 `migrate_to_ci4()` 过程中中断：
- 最坏情况：CI3 表已被删除但新表记录未插完
- 恢复机制：再次访问时 `ci3_migrations_exists()` 返回 false（旧表已删）
- 结果：`latest()` 会尝试重新执行所有迁移
- 风险：早期迁移可能重复执行 → 依赖各迁移自身的幂等性设计

---

## 五、配置补齐逻辑

### 5.1 配置表 `app_config` 结构

```sql
CREATE TABLE `ospos_app_config` (
    `key`   varchar(50)  NOT NULL,
    `value` varchar(500) NOT NULL,
    PRIMARY KEY (`key`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

**关键设计**：`key` 是主键，天然防重复。

### 5.2 配置补齐的三种方式

#### 方式 1：PHP 迁移中 `insertBatch` + `ignore(true)`

示例：[20230412000000_add_missing_config.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/20230412000000_add_missing_config.php#L12-L27)

```php
public function up(): void
{
    $image_values = [
        ['key' => 'account_number',                    'value' => ''],
        ['key' => 'category_dropdown',                 'value' => ''],
        ['key' => 'smtp_host',                         'value' => ''],
        // ... 更多配置项
    ];
    $this->db->table('app_config')->ignore(true)->insertBatch($image_values);
}
```

**特性**：
- `ignore(true)` 生成 `INSERT IGNORE` SQL
- 已存在的 key 不会被覆盖，保护用户自定义值
- 批量插入，性能较好

#### 方式 2：SQL 脚本中 `INSERT IGNORE`

示例：[3.4.2_missing_config_keys.sql](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/sqlscripts/3.4.2_missing_config_keys.sql)

```sql
INSERT IGNORE INTO ospos_app_config (`key`, `value`)
VALUES
    ('msg_msg', ''),
    ('msg_pwd', ''),
    ('smtp_timeout', 5000),
    ('smtp_crypto', 'tls'),
    ('smtp_port', 587),
    ('protocol', 'sendmail');
```

#### 方式 3：运行时兜底默认值

代码路径：[OSPOS.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Config/OSPOS.php#L31-L66)

```php
public function set_settings(): void
{
    $cache = $this->cache->get('settings');
    if ($cache) {
        $this->settings = decode_array($cache);
        return;
    }
    
    try {
        $db = Database::connect();
        if (!$db->tableExists('app_config')) {
            $this->settings = $this->getDefaultSettings();  // 表不存在 → 用默认
            return;
        }
        // ... 从数据库读取配置
    } catch (\Exception $e) {
        $this->settings = $this->getDefaultSettings();  // 读失败 → 用默认
    }
}

private function getDefaultSettings(): array
{
    return [
        'language'      => 'english',
        'language_code' => 'en',
        'company'       => 'Home',
        'barcode_type'  => 'Code39'
    ];
}
```

**兜底时机**：
- 数据库表不存在（全新安装还未执行迁移）
- 数据库连接异常
- 配置查询失败

### 5.3 配置补齐的时机与流程

```
代码版本升级
    ↓
访问 login 页面
    ↓
Login::index()
    ├─ migrate_to_ci4()          # CI3→CI4 转换（如需要）
    ├─ is_latest() = false       # 检测到需要升级
    └─ latest() 开始执行
        ├─ 20170501000000_initial_schema.php
        │   └─ 全新安装才执行：initial_schema.sql 包含 ~80 条初始配置
        ├─ 20170501150000_upgrade_to_3_1_1.php
        │   └─ 3.0.2_to_3.1.1.sql 包含该版本新增配置
        ├─ ... 中间各版本迁移 ...
        ├─ 20200508000000_image_upload_defaults.php
        │   └─ insertBatch 图片上传相关配置
        ├─ 20230412000000_add_missing_config.php
        │   └─ insertBatch 补齐历史遗漏配置
        ├─ 20250716170000_MissingConfigKeys.php
        │   └─ 3.4.2_missing_config_keys.sql 补齐邮件相关配置
        └─ 20260506000000_AddShortcutKeys.php
            └─ insertBatch 快捷键配置
    ↓
迁移全部成功
    ↓
post_controller_constructor 事件
    ↓
Load_config::load_config()
    ├─ 检查 is_latest()，非最新则销毁 session
    ├─ OSPOS::set_settings() 加载配置
    │   └─ 缓存缺失配置项由 getDefaultSettings() 兜底
    └─ 设置语言、时区等
```

### 5.4 配置迁移的幂等性设计原则

1. **永远使用 `INSERT IGNORE`**：不覆盖用户已修改的配置
2. **不做 UPDATE 迁移**：避免强制覆盖用户自定义值
3. **不在 `down()` 中删除必要配置**：回滚时不删除关键配置
4. **运行时兜底**：OSPOS 配置类始终提供默认值，即使迁移未完成也能运行

反例（正确做法）：[20230412000000_add_missing_config.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/20230412000000_add_missing_config.php#L32-L35)
```php
public function down(): void
{
    // No need to remove necessary config values.
}
```

---

## 六、完整升级时序图

```
用户浏览器                          Login 控制器               MY_Migration         CI4 MigrationRunner      数据库
    │                                   │                        │                        │                  │
    │ 访问 /login                       │                        │                        │                  │
    ├──────────────────────────────────>│                        │                        │                  │
    │                                   │  new MY_Migration()    │                        │                  │
    │                                   │───────────────────────>│                        │                  │
    │                                   │  migrate_to_ci4()      │                        │                  │
    │                                   │───────────────────────>│ ci3_migrations_exists()│                  │
    │                                   │                        │───────────────────────>│ 检查表结构        │
    │                                   │                        │                        │─────────────────>│
    │                                   │                        │  is_latest()           │                  │
    │                                   │                        ├───────────────────────>│ findMigrations()  │
    │                                   │                        │                        │ getHistory()      │
    │                                   │  返回视图（需升级）     │                        │                  │
    │<──────────────────────────────────┤                        │                        │                  │
    │                                   │                        │                        │                  │
    │ 点击"迁移"按钮（AJAX POST）        │                        │                        │                  │
    ├──────────────────────────────────>│                        │                        │                  │
    │                                   │  /migrate 接口          │                        │                  │
    │                                   ├───────────────────────>│ migrate_to_ci4()       │                  │
    │                                   │                        │ latest()               │                  │
    │                                   │                        ├───────────────────────>│ ensureTable()     │
    │                                   │                        │                        │ findMigrations()  │
    │                                   │                        │                        │ 过滤已执行迁移     │
    │                                   │                        │                        │ foreach 待执行:    │
    │                                   │                        │                        │   migrate('up')    │
    │                                   │                        │                        │   addHistory()     │
    │                                   │                        │                        │  失败 → regress()  │
    │                                   │                        │                        │  抛出异常          │
    │  返回 JSON {success:true/false}   │                        │                        │                  │
    │<──────────────────────────────────┤                        │                        │                  │
    │                                   │                        │                        │                  │
    │ 前端检测到成功，刷新页面           │                        │                        │                  │
    ├──────────────────────────────────>│  index()               │                        │                  │
    │                                   │  is_latest() = true    │                        │                  │
    │                                   │  显示登录表单           │                        │                  │
    │<──────────────────────────────────┤                        │                        │                  │
```

---

## 七、关键风险点与注意事项

### 7.1 迁移失败的常见原因

1. **超时**：大版本升级涉及大量数据转换（如 CI3→CI4 加密数据转换）
   - 应对：`set_time_limit(3600)`，AJAX 超时设为 3600000ms

2. **SQL 错误**：不兼容的 SQL 语法、外键约束冲突
   - 应对：`execute_script()` 记录详细错误日志，幂等性设计保证可重试

3. **外键约束导致 DDL 失败**：如字符集转换时外键依赖
   - 应对：[upgrade_to_3_1_1.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/20170501150000_upgrade_to_3_1_1.php#L24-L51) 中的模式
     ```php
     $constraints = dropAllForeignKeyConstraints($table, $column);
     execute_script(...);
     recreateForeignKeyConstraints($constraints);
     ```

### 7.2 配置补齐的边界情况

1. **全新安装**：`initial_schema.sql` 一次性插入所有基础配置，后续迁移只补新增项

2. **跨版本升级**：如从 3.2.0 直接升级到 3.4.2
   - 各版本迁移按顺序执行
   - 每个版本的配置补齐只补该版本新增的
   - `INSERT IGNORE` 保证即使前面版本漏了也不会冲突

3. **用户手动删除配置**：
   - 重新执行迁移不会恢复（已存在于 migrations 历史）
   - 需要手动插入或使用 `Appconfig::save()`

### 7.3 性能考量

1. **配置缓存**：[OSPOS.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Config/OSPOS.php#L33-L53) 中使用缓存避免每次请求读数据库
   ```php
   $cache = $this->cache->get('settings');
   if ($cache) {
       $this->settings = decode_array($cache);
       return;
   }
   ```

2. **迁移后缓存失效**：[Appconfig.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Models/Appconfig.php#L77-L90) 的 `save()` 会自动清除缓存
   ```php
   public function save($data): bool
   {
       $success = parent::save($save_data);
       if ($success) {
           config(OSPOS::class)->update_settings();  // 清除缓存 + 重新加载
       }
       return $success;
   }
   ```

---

## 八、代码路径索引

### 版本管理
- 版本读取：[MY_Migration::get_current_version()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Libraries/MY_Migration.php#L36-L53)
- 最新版本：[MY_Migration::get_latest_migration()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Libraries/MY_Migration.php#L25-L29)
- 版本比较：[MY_Migration::is_latest()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Libraries/MY_Migration.php#L14-L20)

### 迁移执行
- 入口检测：[Login::index()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Controllers/Login.php#L23-L57)
- AJAX 接口：[Login::migrate()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Controllers/Login.php#L77-L99)
- CI3→CI4 转换：[MY_Migration::migrate_to_ci4()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Libraries/MY_Migration.php#L58-L64)
- SQL 执行：[migration_helper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Helpers/migration_helper.php)

### 中断恢复
- 批次回滚：CI4 `MigrationRunner::regress()`
- 幂等插入：`$builder->ignore(true)->insertBatch()`
- 存在性检查：[migration_helper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Helpers/migration_helper.php#L223-L237) 中 `indexExists()`, `foreignKeyExists()`, `primaryKeyExists()`

### 配置补齐
- 初始配置：[initial_schema.sql](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/sqlscripts/initial_schema.sql#L6-L85)
- 配置模型：[Appconfig.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Models/Appconfig.php)
- 配置加载：[OSPOS::set_settings()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Config/OSPOS.php#L31-L56)
- 默认值兜底：[OSPOS::getDefaultSettings()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Config/OSPOS.php#L58-L66)
- 配置迁移示例：
  - [20200508000000_image_upload_defaults.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/20200508000000_image_upload_defaults.php)
  - [20230412000000_add_missing_config.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/20230412000000_add_missing_config.php)
  - [20260506000000_AddShortcutKeys.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/20260506000000_AddShortcutKeys.php)
  - [3.4.2_missing_config_keys.sql](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/sqlscripts/3.4.2_missing_config_keys.sql)
