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

### 5.5 配置读入与缓存刷新机制

#### 5.5.1 配置实例化与加载时机

**CI4 配置工厂的单例特性**：
`config(OSPOS::class)` 是 CI4 的配置工厂函数，同一请求内多次调用返回**同一实例**。但每次 HTTP 请求会创建新实例。

**完整加载流程**（[OSPOS.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Config/OSPOS.php)）：

```
请求开始
    ↓
任意代码首次调用 config(OSPOS::class)
    ↓
new OSPOS() 实例化
    ├─ $this->cache = Services::cache()  // 获取缓存服务
    └─ $this->set_settings()             // 立即调用加载方法
        ├─ $cache = $this->cache->get('settings')  // 尝试读缓存
        ├─ 缓存命中 → decode_array() → $this->settings  // 直接返回
        └─ 缓存未命中
            ├─ 连接数据库 → 查询 app_config 表
            ├─ 遍历结果构建 $this->settings 数组
            ├─ encode_array() → $this->cache->save('settings', ...)  // 写入缓存
            └─ 异常时 → getDefaultSettings() 兜底
```

**关键点**：
- `set_settings()` 只在**构造函数**中调用一次
- 同一请求内配置加载后，后续访问 `$config->settings` 直接读取内存数组
- 不同请求之间配置互相独立

#### 5.5.2 缓存刷新的触发点

**手动刷新**：[OSPOS::update_settings()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Config/OSPOS.php#L71-L75)
```php
public function update_settings(): void
{
    $this->cache->delete('settings');  // 删除缓存
    $this->set_settings();             // 重新加载
}
```

**自动刷新**：[Appconfig::save()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Models/Appconfig.php#L77-L90)
```php
public function save($data): bool
{
    $success = parent::save($save_data);
    if ($success) {
        config(OSPOS::class)->update_settings();  // 保存成功自动刷新
    }
    return $success;
}
```

**缓存配置**：[Cache.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Config/Cache.php#L25-L87)
- 默认使用 `file` 缓存驱动
- 缓存目录：`WRITEPATH . 'cache/'`
- 默认 TTL：300 秒（5 分钟）

#### 5.5.3 迁移后配置生效时机

**两种配置补齐方式的生效差异**：

| 补齐方式 | 数据库写入 | 缓存刷新 | 当前请求可见 | 下请求可见 |
|----------|------------|----------|--------------|------------|
| SQL `INSERT IGNORE`（直接执行） | ✅ 立即写入 | ❌ 不刷新 | ❌ 不可见（内存中还是旧值） | ⚠️ 取决于缓存是否过期 |
| `Appconfig::save()`（走模型） | ✅ 立即写入 | ✅ 自动刷新 | ✅ 可见（单例模式重新加载） | ✅ 可见 |

**场景分析 1：SQL 直接插入配置**

以 [20250716170000_MissingConfigKeys.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/20250716170000_MissingConfigKeys.php) 为例：
```php
// 执行 SQL 脚本，直接 INSERT IGNORE
executeScriptWithTransaction(APPPATH . 'Database/Migrations/sqlscripts/3.4.2_missing_config_keys.sql');
```

**生效时序**：
```
T1: 请求开始 → OSPOS 实例化 → set_settings() → 加载旧配置到内存
T2: latest() 开始执行
T3: 迁移 1: executeScriptWithTransaction() → INSERT IGNORE 配置到数据库
    ├─ 数据库：新配置已写入 ✅
    ├─ 缓存：未变化 ❌
    └─ 当前请求 OSPOS 实例：内存中无新配置 ❌
T4: 迁移 2: ...（其他迁移）
T5: 迁移全部完成 → redirect()->to('login')  // 触发新请求
T6: 新请求开始 → 新 OSPOS 实例化 → set_settings()
    ├─ 检查缓存：
    │   ├─ 如果缓存未过期（<5分钟）→ 读到旧缓存 → 无新配置 ❌
    │   └─ 如果缓存已过期 → 重新读数据库 → 有新配置 ✅
    └─ 构建 settings 数组
```

**问题**：迁移完成后，如果缓存还在 5 分钟有效期内，用户可能在一段时间内读不到新配置。

**场景分析 2：通过 Appconfig::save() 更新配置**

以 [20240319000000_Migration_Convert_Barcode_Types.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/20240319000000_Migration_Convert_Barcode_Types.php#L47) 为例：
```php
$this->appconfig->save(['barcode_type' => $new_barcode_type]);
```

**生效时序**：
```
T1: 请求开始 → OSPOS 实例化 → 加载配置
T2: 迁移执行 save(['barcode_type' => 'C39'])
    ├─ 数据库：更新成功 ✅
    └─ 触发 update_settings()
        ├─ cache->delete('settings')  // 清除缓存 ✅
        └─ set_settings() 重新加载
            ├─ 缓存已删除 → 从数据库重新读 ✅
            └─ 当前实例 $this->settings 已更新 ✅
T3: 同一请求内后续代码访问 config(OSPOS::class)->settings['barcode_type']
    └─ 单例模式 → 已有更新后的值 ✅
T4: 迁移完成 → 重定向
T5: 新请求 → 缓存已清除 → 从数据库读 → 最新配置 ✅
```

**实际保障机制**：[Load_config::load_config()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Events/Load_config.php#L33-L35)
```php
if (!$migration->is_latest()) {
    $this->session->destroy();  // 非最新版本时销毁 session
}
```

迁移过程中 `is_latest()` 返回 false，session 被销毁。迁移完成后重定向：
- 新请求中 `is_latest()` 返回 true
- 但配置缓存可能仍在有效期内（5分钟）
- **关键问题**：OSPOS 类没有监听迁移完成事件来主动刷新缓存

---

## 六、迁移失败回滚与整批重跑边界

### 6.1 CI4 `latest()` 执行的原子性分析

**核心时序**（基于 CI4 `MigrationRunner::latest()` 源码）：
```
foreach ($migrations as $migration) {
    if ($this->migrate('up', $migration)) {   // 1. 执行迁移 up()
        $this->addHistory($migration, $batch); // 2. 写入 migrations 表
    } else {
        $this->regress(-1);                   // 3. 失败 → 回滚整批
        throw new RuntimeException(...);
    }
}
```

**三个时间窗口的风险**：

| 时间点 | 状态 | 中断后果 | 恢复机制 |
|--------|------|----------|----------|
| T1: `migrate('up')` 执行中 | 数据库数据已变更，migrations 表未更新 | 数据已写但无历史记录 | 下次 `latest()` 重新执行该迁移，依赖幂等性 |
| T2: `addHistory()` 执行中 | 数据已变更，正在写入历史 | 历史记录可能不完整 | 下次 `latest()` 可能重复执行 |
| T3: `addHistory()` 成功后 | 数据和历史都已写入 | 下一个迁移开始 | 正常，下次跳过已记录迁移 |

### 6.2 `regress(-1)` 回滚的边界限制

**回滚流程**：
```
regress(-1) 被调用
    ↓
获取当前批次 batch=N 的所有历史记录（按倒序）
    ↓
foreach 历史记录（倒序）:
    ├─ 实例化迁移类
    ├─ 调用 $migration->down()
    └─ 成功 → removeHistory() 删除历史记录
```

**能回滚的**：
1. 迁移类 `down()` 方法中有实际回滚逻辑的
2. `migrations` 表中的历史记录（CI4 自动删除）

**不能回滚的（关键边界）**：

#### 边界 1：空 `down()` 方法

绝大多数配置迁移的 `down()` 是空实现：
```php
// 20250716170000_MissingConfigKeys.php
public function down(): void {}  // 配置不会被删除

// 20230412000000_add_missing_config.php
public function down(): void
{
    // No need to remove necessary config values.  // 明确不删除
}
```

**后果**：回滚时配置数据**仍留在数据库中**，但 `migrations` 表中的历史记录被删除。下次重试时：
- 配置已存在 → `INSERT IGNORE` 跳过（无错误）
- 迁移重新执行成功 → 重新写入历史记录
- ✅ 最终一致性

#### 边界 2：有实际逻辑但不可逆的 `down()`

```php
// 20250521000000_FixImageFilenameSpaces.php
public function down(): void
{
    // This migration cannot be safely reversed as the original filenames are lost
    // after sanitization.  // 文件名已丢失，无法回滚
}
```

#### 边界 3：MySQL DDL 隐式提交

MySQL 中 DDL 语句（CREATE, ALTER, DROP, TRUNCATE 等）执行后会**隐式提交**当前事务，无法回滚。

```php
// migration_helper.php 中的 execute_script()
foreach ($sql as $statement) {
    $this->db->query($statement);  // DDL 语句执行后自动提交
    // 即使后面用了 transRollback()，DDL 也不会回滚
}
```

**后果**：
- `executeScriptWithTransaction()` 中的 DDL 也无法回滚
- 迁移失败时，表结构可能已经变更
- 依赖幂等性检查（`indexExists()`, `fieldExists()` 等）保证重试安全

#### 边界 4：数据删除无法恢复

```php
// 20200508000000_image_upload_defaults.php
public function down(): void
{
    $builder = $this->db->table('app_config');
    $builder->whereIn('key', ['image_allowed_types', ...]);
    $builder->delete();  // 回滚时删除配置
}
```

**风险**：如果用户在迁移执行后修改了这些配置值，回滚时会被删除且无法恢复。

### 6.3 失败后整批重跑的三种场景

#### 场景 1：迁移 C 失败，触发完整回滚

```
批次 N 开始
  ├─ 迁移 A up() 成功 → addHistory(batch=N) ✓
  ├─ 迁移 B up() 成功 → addHistory(batch=N) ✓
  └─ 迁移 C up() 失败 ✗
      ├─ regress(-1) 触发
      ├─ 查询 batch=N → 找到 A、B
      ├─ B.down() → removeHistory(B)
      └─ A.down() → removeHistory(A)
```

**重跑时**：
- `getHistory()` 查询不到 A、B、C 的记录
- A、B、C 都会被重新执行
- ✅ 依赖 A、B 的 `up()` 幂等性

#### 场景 2：迁移 B up() 成功，但 `addHistory()` 之前中断

```
批次 N 开始
  ├─ 迁移 A up() → addHistory() ✓
  └─ 迁移 B up() 成功（数据已写）
     └─ 在 addHistory(B) 之前中断（如 PHP 致命错误、服务器重启）
```

**状态**：
- 数据库：迁移 B 的数据变更已生效
- migrations 表：只有 A 的记录，没有 B 的记录

**重跑时**：
- `getHistory()` 只返回 A
- 迁移 A 被跳过（已在历史中）
- 迁移 B 被重新执行
- ✅ 依赖 B 的 `up()` 幂等性（`INSERT IGNORE`、存在性检查等）

#### 场景 3：迁移 B `addHistory()` 成功后中断

```
批次 N 开始
  ├─ 迁移 A up() → addHistory() ✓
  ├─ 迁移 B up() → addHistory() ✓
  └─ 在执行迁移 C 之前中断
```

**状态**：
- 数据库：A、B 的数据变更已生效
- migrations 表：A、B 的记录都在，batch=N

**重跑时**：
- `getHistory()` 返回 A、B
- A、B 被跳过
- 只执行 C 和后续迁移
- ✅ 最理想情况，无重复执行

### 6.4 CI3→CI4 转换的特殊边界

[MY_Migration::migrate_table()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Libraries/MY_Migration.php#L84-L110)

```php
private function migrate_table(string $ci3_migrations_version): void
{
    $this->convert_table();  // 1. DROP TABLE IF EXISTS migrations; CREATE TABLE ...
    // 2. 循环插入历史记录
    foreach ($available_migrations as $version => $path) {
        if ($version > (int)$ci3_migrations_version) break;
        $this->addHistory($migration, 1);
    }
}
```

**中断风险**：
- T1: `convert_table()` 执行成功 → 旧表已删，新表已建但为空
- T2: 循环插入历史记录时中断
- **后果**：新表中只有部分历史记录

**重跑时**：
- `ci3_migrations_exists()` 返回 false（旧表已删，新表有 id 字段）
- 跳过 `migrate_to_ci4()`
- `latest()` 开始执行
- 对于已写入历史的迁移 → 跳过
- 对于未写入历史但 CI3 时代已执行的迁移 → 重新执行
- ⚠️ 依赖所有迁移的幂等性

### 6.5 幂等性设计的关键模式

**必须遵循的模式**：

1. **配置插入**：永远用 `INSERT IGNORE`
   ```php
   $this->db->table('app_config')->ignore(true)->insertBatch($values);
   ```

2. **索引/外键添加**：先检查后操作
   ```php
   if (!foreignKeyExists($table, $constraintName)) {
       $this->db->query('ALTER TABLE ... ADD CONSTRAINT ...');
   }
   ```

3. **列添加**：用 `$this->forge->addColumn()` 或检查字段存在
   ```php
   if (!$this->db->fieldExists('new_column', $table)) {
       $this->forge->addColumn($table, ['new_column' => ['type' => 'INT']]);
   }
   ```

4. **数据转换**：用事务包裹，失败回滚
   ```php
   executeScriptWithTransaction($sql_file);  // 纯数据操作可用事务
   ```

---

## 七、完整升级时序图

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

## 八、AJAX 迁移成功后的页面处理：同页、下请求与回滚重算

### 8.1 两条迁移入口与 JS 分流逻辑

OSPOS 有两条触发迁移的代码路径，由前端 JS 根据 `APP_STATE.isNewInstall` 分流。

#### 分流逻辑

[login.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Views/login.php#L257-L288)

```javascript
$form.on('submit', function(e) {
    if (APP_STATE.isNewInstall) {   // ← 分流点
        e.preventDefault();         // 阻止表单提交
        // ... AJAX 调用 /migrate
    }
    // APP_STATE.isNewInstall = false 时，不拦截 → 表单正常 POST /login
});
```

`APP_STATE.isNewInstall` 的值由 PHP 模板注入：

[login.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Views/login.php#L171)

```javascript
isNewInstall: <?= $is_new_install ? 'true' : 'false' ?>,
```

`$is_new_install` 来自 [Login::index()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Controllers/Login.php#L40)：

```php
$is_new_install = !(MY_Migration::get_current_version());
// 数据库 migrations 表为空或不存在 → true（全新安装）
// 数据库有迁移记录 → false（升级场景）
```

**因此两条路径的区分**：
- **全新安装**：`is_new_install = true` → JS 拦截表单 → AJAX POST `/migrate`
- **升级场景**：`is_new_install = false` → JS 不拦截 → 表单 POST `/login`

> 注意：升级场景虽然 `is_latest = false`，但 `is_new_install = false`，
> 所以 JS **不拦截**表单提交，走的是 `Login::index()` 中的 POST 分支。

### 8.2 路径 A：升级场景 — 表单 POST `/login`

[Login::index()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Controllers/Login.php#L48-L57)

```php
if ($this->request->getMethod() !== 'POST') {
    return view('login', $data);  // GET → 显示登录页
}

if (!$data['is_latest'] || $data['is_new_install']) {
    set_time_limit(3600);
    $migration->setNamespace('App')->latest();
    return redirect()->to('login');  // 302 重定向 → 新 GET 请求
}
```

**完整请求链**：

```
1. GET /login           → 渲染页面，显示升级提示
2. POST /login（登录）   → 服务端执行 latest()
   ├─ 成功 → 302 redirect → GET /login → 新请求，全新加载配置
   └─ 失败 → 抛异常，PHP 白屏（无优雅处理）
3. GET /login（重定向后）→ is_latest=true → 显示登录表单
```

**特点**：
- 迁移成功后由 `redirect()->to('login')` 触发浏览器 302 跳转
- 新的 GET 请求中 OSPOS 实例重新创建，配置从缓存/数据库重新加载
- ✅ **不存在同页旧配置问题**，因为页面完全刷新

### 8.3 路径 B：全新安装 — AJAX POST `/migrate`

[login.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Views/login.php#L257-L288)

```javascript
if (APP_STATE.isNewInstall) {
    e.preventDefault();
    showMigrationProgress();

    $.ajax({
        url: APP_STATE.migrateUrl,   // POST /migrate
        timeout: 3600000,
        data: { [APP_STATE.csrfToken]: APP_STATE.csrfHash },
        success: function(response) {
            if (response.success) {
                APP_STATE.isNewInstall = false;   // ← 改 JS 状态变量
                showMigrationSuccess();            // ← 同页 DOM 操作
            } else {
                showMigrationError(response.message);
            }
        },
        error: function(xhr) {
            showMigrationError(message);
        }
    });
}
```

[Login::migrate()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Controllers/Login.php#L77-L99)

```php
public function migrate(): ResponseInterface
{
    try {
        $migration = new MY_Migration(config('Migrations'));
        $migration->migrate_to_ci4();
        set_time_limit(3600);
        $migration->setNamespace('App')->latest();
        return $this->response->setJSON(['success' => true, ...]);
    } catch (\Exception $e) {
        return $this->response->setJSON(['success' => false, ...])->setStatusCode(500);
    }
}
```

**完整请求链**：

```
1. GET /login           → 渲染页面，is_new_install=true，显示"迁移"按钮
2. 点击"迁移" → AJAX POST /migrate
   ├─ 成功 → JSON {success:true} → showMigrationSuccess()（同页 DOM 操作）
   └─ 失败 → JSON {success:false} → showMigrationError()（同页 DOM 操作）
3. 用户输入密码，点击"Go" → 表单正常 POST /login
   （APP_STATE.isNewInstall 已被改为 false，JS 不再拦截）
4. POST /login → is_latest=true → 验证登录 → redirect→home
```

### 8.4 同页不刷新的精确分析

**`showMigrationSuccess()` 做了什么**：

[login.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Views/login.php#L221-L230)

```javascript
function showMigrationSuccess() {
    $progress.addClass('d-none');        // 隐藏进度条
    $error.addClass('d-none');           // 隐藏错误框
    $warning.addClass('d-none');         // 隐藏升级提示
    $success.removeClass('d-none');      // 显示"迁移完成"提示
    $heading.text(APP_STATE.i18n.welcome);  // 标题改为 "欢迎"
    $loginFields.removeClass('d-none');  // 显示登录表单
    $submitButton.text(APP_STATE.i18n.go);  // 按钮文字改为 "Go"
    $submitButton.prop('disabled', false);
}
```

**页面没有刷新**，但理解"旧 DOM"的影响需要区分两层：

#### 第一层：PHP 硬编码到 HTML 的配置值（确实没更新）

这些是首次 GET 渲染时由 PHP 写入 HTML 的值，AJAX 成功后不会变：

| DOM 元素 | 来源 | 全新安装时的初始值 | AJAX 后 | 影响评估 |
|----------|------|--------------------|---------|----------|
| `<html lang>` | [current_language_code()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Views/login.php#L19) | `en`（默认值） | ❌ 不变 | ⚠️ 无影响：登录页无多语言内容 |
| `<title>` | [$config['company']](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Views/login.php#L24) | `Home`（默认值） | ❌ 不变 | ⚠️ 无影响：标题不影响功能 |
| 主题 CSS | [$config['theme']](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Views/login.php#L29-L35) | `flatly`（默认值） | ❌ 不变 | ⚠️ 无影响：默认主题就是新安装主题 |
| 登录表单样式 | [$config['login_form']](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Views/login.php#L99) | `floating_labels`（默认） | ❌ 不变 | ⚠️ 无影响：默认值与新安装一致 |
| reCAPTCHA | [$config['gcaptcha_site_key']](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Views/login.php#L131) | 空字符串（默认） | ❌ 不变 | ✅ 无影响：新安装不启用 reCAPTCHA |
| Logo | [$config['company_logo']](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Views/login.php#L44) | 空（默认）→ 显示 SVG | ❌ 不变 | ⚠️ 无影响：默认就是 SVG logo |

**关键纠正**：说"页面停留在旧 DOM"容易误导。对于全新安装场景，首次渲染时数据库还是空的，
PHP 使用的是 `getDefaultSettings()` 返回的 4 个默认值（`language=english`, `language_code=en`,
`company=Home`, `barcode_type=Code39`）。迁移后数据库写入的配置值与默认值大部分一致
（公司名仍是 Home，主题仍是 flatly），**不存在"旧值"与"新值"冲突的问题**。

#### 第二层：JS 动态管理的 DOM 元素（已正确更新）

| DOM 元素 | 更新方式 | 值来源 |
|----------|----------|--------|
| 标题 `<h3>` | `$heading.text(APP_STATE.i18n.welcome)` | JS 内置翻译字符串，不依赖服务端配置 |
| 按钮文字 | `$submitButton.text(APP_STATE.i18n.go)` | JS 内置翻译字符串 |
| 成功提示 | `$success.removeClass('d-none')` | PHP 硬编码在 HTML 中，翻译字符串 |

**这些元素完全由 JS 客户端状态驱动，与服务端配置无关，不涉及旧值问题。**

### 8.5 登录提交后的配置生效时机

AJAX 成功后用户输入密码点击 "Go"：

```
点击 "Go" → 表单提交
  ├─ APP_STATE.isNewInstall = false（已被 AJAX 成功回调修改）
  ├─ if (APP_STATE.isNewInstall) 不成立 → 不拦截
  └─ 表单正常 POST /login
```

**服务端 `Login::index()` 处理此 POST 请求的精确路径**：

[Login::index()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Controllers/Login.php#L23-L75)

```php
// 步骤 1：创建迁移实例
$migration = new MY_Migration(config('Migrations'));
$config = config(OSPOS::class)->settings;  // ← 此时加载配置

// 步骤 2：检查方法
if ($this->request->getMethod() !== 'POST') {
    return view('login', $data);  // GET → 不走这里
}

// 步骤 3：检查是否需要迁移
if (!$data['is_latest'] || $data['is_new_install']) {
    // is_latest = true, is_new_install = false → 不走这里
}

// 步骤 4：验证登录
$rules = ['username' => 'required|login_check[data]'];
if (!$this->validate($rules, $messages)) {
    return view('login', $data);  // 验证失败 → 重新渲染页面
}

// 步骤 5：登录成功
return redirect()->to('home');
```

**配置在这个 POST 请求中的加载链**：

```
POST /login 请求开始
  ├─ CI4 框架启动
  ├─ 控制器实例化 → post_controller_constructor 事件
  │   └─ Load_config::load_config()
  │       ├─ new MY_Migration() → is_latest() = true → 不销毁 session ✅
  │       ├─ config(OSPOS::class)->settings  ← 此时 OSPOS 已实例化
  │       │   └─ set_settings()
  │       │       ├─ 读缓存：cache->get('settings')
  │       │       │   ├─ 缓存命中（未过期）→ 使用缓存值
  │       │       │   └─ 缓存未命中 → 查数据库 → 写缓存
  │       │       └─ 设置语言、时区
  │       └─ 使用配置值设置运行时环境
  │
  ├─ Login::index() 执行
  │   ├─ config(OSPOS::class)->settings  ← 同一单例
  │   ├─ is_latest() = true → 不执行迁移
  │   └─ 验证登录
  │
  └─ 登录成功 → redirect()->to('home')
      └─ GET /home → 新请求 → 新 OSPOS 实例 → 重新加载配置
```

**缓存状态的关键时间窗口**：

| 迁移中的操作 | 缓存效果 | 后续请求读到的配置 |
|-------------|----------|-------------------|
| `$this->db->table('app_config')->ignore(true)->insertBatch(...)` | 缓存**未刷新** | ⚠️ 缓存若未过期则读到旧值 |
| `$this->db->query("INSERT IGNORE INTO app_config ...")` via SQL 文件 | 缓存**未刷新** | ⚠️ 同上 |
| `$appconfig->save(['key' => 'value'])` | 缓存**已刷新**（`update_settings()`） | ✅ 读到最新值 |
| `executeScriptWithTransaction($sql_file)` | 缓存**未刷新** | ⚠️ 同上 |

**但实际影响有限**：

1. **全新安装场景**：首次 GET 时缓存中存的是 `getDefaultSettings()` 的 4 个默认值
   或空配置。迁移执行后 `initial_schema.sql` 写入大量配置到数据库，
   但缓存**不会自动清除**。不过——
   
2. **缓存中存的是什么**：首次 GET 时 `app_config` 表不存在（全新安装），
   所以 `set_settings()` 走了 `$this->settings = $this->getDefaultSettings()` 分支，
   **但没有调用 `cache->save()`**——看代码：

   [OSPOS::set_settings()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Config/OSPOS.php#L40-L46)
   ```php
   if (!$db->tableExists('app_config')) {
       $this->settings = $this->getDefaultSettings();
       return;  // ← 直接返回，没有写缓存！
   }
   ```

3. **因此缓存中没有旧配置条目**：全新安装时缓存从未写入 `settings` 键。
   迁移后的 POST 请求中 `cache->get('settings')` 返回 false → 走数据库读取 → ✅ 最新配置

4. **升级场景不经过 AJAX 路径**：已确认升级走路径 A（POST /login + redirect），
   新 GET 请求中 OSPOS 重新实例化。缓存可能还是旧的，但——
   如果迁移中有任何 `Appconfig::save()` 调用，缓存已被清除。

**精确结论**：

| 场景 | 缓存状态 | 配置是否最新 |
|------|----------|-------------|
| 全新安装 AJAX 成功 → POST 登录 | 缓存无 `settings` 键（从未写入） | ✅ 最新（从数据库读） |
| 升级路径 A + 迁移中有 `save()` | 缓存已被 `save()` 清除 | ✅ 最新（从数据库读） |
| 升级路径 A + 迁移全是 SQL 插入 | 缓存仍是旧值（5分钟内） | ⚠️ 可能是旧值 |
| 升级路径 A + 缓存已过期（>5分钟） | 缓存过期自动重新读 | ✅ 最新（从数据库读） |

### 8.6 失败后批次回滚重算

#### 8.6.1 AJAX 失败的前端处理

[login.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Views/login.php#L232-L241)

```javascript
function showMigrationError(message) {
    $progress.addClass('d-none');
    $success.addClass('d-none');
    $loginFields.addClass('d-none');
    $errorMessage.text(message);
    $error.removeClass('d-none');
    $warning.addClass('d-none');
    $submitButton.text(APP_STATE.i18n.migrate);  // 按钮恢复为 "迁移"
    $submitButton.prop('disabled', false);        // 重新可点击
}
```

**关键**：失败时 `APP_STATE.isNewInstall` 仍为 `true`（成功回调中才改为 `false`）。
用户再次点击按钮 → `if (APP_STATE.isNewInstall)` 仍成立 → 再次 AJAX POST `/migrate`。

#### 8.6.2 服务端重试：`latest()` 重算批次

第二次调用 `Login::migrate()` → `latest()`，CI4 的 `latest()` 重新执行完整的
"扫描 → 对比历史 → 执行缺失" 流程：

```
latest() 重算过程：
  ├─ ensureTable() → migrations 表已存在，跳过
  ├─ findMigrations() → 扫描所有迁移文件，按版本升序排列
  ├─ getHistory($group) → 查询 migrations 表中已执行记录
  ├─ 过滤：移除历史中已有的迁移 → 得到待执行列表
  ├─ getLastBatch() + 1 → 新批次号
  └─ foreach 待执行 → 逐个 migrate('up') → addHistory()
```

**重算的精确逻辑**取决于首次失败时 migrations 表的状态：

#### 场景 1：`regress(-1)` 正常回滚后重算

```
首次 latest() 执行：
  ├─ 迁移 A: up() 成功 → addHistory(batch=2) ✓
  ├─ 迁移 B: up() 成功 → addHistory(batch=2) ✓
  └─ 迁移 C: up() 抛异常 ✗
      ├─ regress(-1) 自动触发
      ├─ B.down() → removeHistory(B) → 删除 batch=2 中 B 的记录
      ├─ A.down() → removeHistory(A) → 删除 batch=2 中 A 的记录
      └─ 抛出 RuntimeException → 被 catch 捕获

migrations 表状态：batch=2 的记录全部被清除
```

**第二次 `latest()` 重算**：
- `getHistory()` 不含 A、B → A、B 都在待执行列表中
- `getLastBatch()` = 1（假设之前完成到 batch=1）→ 新 batch=2
- A、B、C 全部重新执行，A、B 依赖幂等性
- ✅ 最终一致

#### 场景 2：PHP 致命错误（`regress` 未执行）

```
首次 latest() 执行：
  ├─ 迁移 A: up() 成功 → addHistory(batch=2) ✓
  └─ 迁移 B: up() 中 PHP Fatal Error
      ├─ B 的数据可能已部分写入数据库
      ├─ addHistory(B) 未执行
      └─ regress(-1) 未执行 → 进程崩溃

migrations 表状态：只有 A 的记录（batch=2）
```

**第二次 `latest()` 重算**：
- `getHistory()` 返回 A（batch=2）→ A 被过滤掉
- B 不在历史中 → B 在待执行列表中
- `getLastBatch()` = 2 → 新 batch=3
- 只重新执行 B（⚠️ 依赖 B 的幂等性）
- A 留在 batch=2，B、C 记录在 batch=3

#### 场景 3：`addHistory()` 成功后、下一个迁移之前中断

```
首次 latest() 执行：
  ├─ 迁移 A: up() 成功 → addHistory(batch=2) ✓
  ├─ 迁移 B: up() 成功 → addHistory(batch=2) ✓
  └─ （服务器重启 / 网络断开，在执行 C 之前）

migrations 表状态：A、B 都在 batch=2
```

**第二次 `latest()` 重算**：
- `getHistory()` 返回 A、B → 都被过滤掉
- 只执行 C 和后续迁移
- ✅ 最理想情况，无重复执行

#### 8.6.3 `execute_script()` 的静默失败：一个特殊边界

[execute_script()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Helpers/migration_helper.php#L10-L42)

```php
function execute_script(string $path): bool
{
    $success = true;
    foreach ($sqls as $statement) {
        $hadError = !$db->simpleQuery($statement);
        if ($hadError) {
            $success = false;        // 记录失败
            // 但不抛异常，继续执行后续 SQL
        }
    }
    return $success;
}
```

**问题**：`execute_script()` 返回 `false`，但迁移类的 `up()` 方法**不检查返回值**：
```php
public function up(): void
{
    helper('migration');
    execute_script($sql_file);  // 返回值被忽略
}
```

**后果**：`up()` 正常返回（void），CI4 认为迁移成功，调用 `addHistory()` 写入历史。
部分 SQL 失败但迁移被标记为成功。这是 `execute_script()` 的设计选择——容错继续执行，
避免 DDL 语句的事务问题。这意味着 `regress(-1)` **不会触发**，因为 CI4 不知道失败了。

**对比**：[executeScriptWithTransaction()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Helpers/migration_helper.php#L49-L88) 会在 SQL 失败时 `transRollback()` 回滚 DML 操作，
但返回值同样未被 `up()` 检查。不过事务回滚保证了数据库一致性。

#### 8.6.4 `down()` 回滚边界对重跑的影响

| `down()` 类型 | 回滚行为 | 重跑结果 | 示例 |
|--------------|----------|---------|------|
| **空实现** | 配置保留，历史删除 | `INSERT IGNORE` 跳过 → ✅ | [20250716170000](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/20250716170000_MissingConfigKeys.php#L21-L24) |
| **不删除配置** | 配置保留，历史删除 | `INSERT IGNORE` 跳过 → ✅ | [20230412000000](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/20230412000000_add_missing_config.php#L32-L35) |
| **删除配置** | 配置删除，历史删除 | 重新插入 → ✅ | [20200508000000](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/20200508000000_image_upload_defaults.php#L29-L34) |
| **通过 save() 更新** | save() 刷新缓存，值回退 | 重新 save() → ✅ | [20240319000000](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/20240319000000_Migration_Convert_Barcode_Types.php#L53-L75) |
| **DDL 变更** | DDL 无法回滚 | 存在性检查跳过 → ✅ | 各版本 DDL 迁移 |
| **不可逆** | down() 为空 | 幂等 up() → ✅ | [20250521000000](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/20250521000000_FixImageFilenameSpaces.php#L60-L64) |

**结论**：无论 `down()` 实现如何，重跑都能成功——这是 OSPOS 迁移幂等性设计的核心保障。

#### 8.6.5 `regress(-1)` 本身失败的边界

```
latest() 执行中
  ├─ 迁移 A: up() 成功 → addHistory(batch=2) ✓
  └─ 迁移 B: up() 失败 ✗
      ├─ regress(-1) 触发
      ├─ A.down() 失败（如 DDL 无法回滚）
      │   └─ removeHistory(A) 不执行 → A 的历史记录仍在
      └─ 抛出 RuntimeException
```

**第二次 `latest()` 重算**：
- `getHistory()` 返回 A（batch=2）→ A 被过滤掉
- 只重新执行 B
- ⚠️ A 的回滚未完成，但 A 的 `up()` 已生效
- 后续迁移依赖 A 的变更时不会出错（A 的效果已存在）

### 8.7 完整的 AJAX 迁移生命周期图（纠正版）

```
┌──────────────────────────────────────────────────────────────────────┐
│ 1. GET /login（首次访问）                                             │
│                                                                      │
│  Login::index()                                                      │
│    ├─ config(OSPOS::class)->settings                                 │
│    │   └─ app_config 表不存在 → getDefaultSettings() → 4 个默认值    │
│    │       ⚠️ 不写缓存（因为 tableExists 检查失败后直接 return）      │
│    ├─ is_latest() = false                                            │
│    ├─ is_new_install = true（migrations 表为空）                      │
│    └─ return view('login', $data)                                    │
│                                                                      │
│  post_controller_constructor → Load_config::load_config()            │
│    ├─ is_latest() = false → session->destroy()                       │
│    └─ 用默认配置设置语言/时区                                         │
│                                                                      │
│  浏览器收到 HTML：                                                    │
│    ├─ <html lang="en">（默认值）                                      │
│    ├─ <title>Home | OSPOS | Login</title>（默认值）                   │
│    ├─ flatly 主题 CSS（默认值）                                       │
│    ├─ 登录表单初始隐藏（is_new_install=true → d-none）                │
│    └─ 按钮显示"迁移"                                                  │
└──────────────────────────────────────────────────────────────────────┘
         │
         │ 用户点击"迁移"
         ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 2. AJAX POST /migrate                                                │
│                                                                      │
│  Login::migrate()                                                    │
│    ├─ new MY_Migration()                                             │
│    ├─ migrate_to_ci4() → 全新安装无需转换                             │
│    ├─ latest() 执行所有迁移                                          │
│    │   ├─ initial_schema.sql → 建表 + 插入 ~80 条初始配置            │
│    │   ├─ ... 各版本迁移 ...                                         │
│    │   └─ 最后一个迁移完成 → addHistory()                             │
│    └─ return JSON {success: true}                                    │
│                                                                      │
│  ⚠️ 此请求的 OSPOS 实例：                                            │
│     - 构造时 app_config 不存在 → getDefaultSettings()                │
│     - 迁移执行后 app_config 已有数据，但 OSPOS 实例不刷新             │
│     - 无影响：此请求只返回 JSON，不渲染 HTML                          │
└──────────────────────────────────────────────────────────────────────┘
         │
         │ JS 收到 {success: true}
         ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 3. 同页 DOM 操作（无新 HTTP 请求）                                    │
│                                                                      │
│  showMigrationSuccess()                                              │
│    ├─ 隐藏进度条、警告、错误                                         │
│    ├─ 显示"迁移完成"提示                                             │
│    ├─ 标题改为 "Welcome to OSPOS"（JS 翻译字符串，非服务端配置）       │
│    ├─ 显示登录表单                                                   │
│    ├─ APP_STATE.isNewInstall = false                                 │
│    └─ 按钮文字改为 "Go"                                              │
│                                                                      │
│  ⚠️ HTML 中 PHP 硬编码的配置值未更新，但：                            │
│     - 全新安装时这些值就是默认值（en/Home/flatly/floating_labels）    │
│     - 迁移后数据库中的配置与默认值一致（initial_schema 使用相同默认值）│
│     - 所以"旧值"= "新值"，不存在不一致                                │
│                                                                      │
│  ✅ 此页面完全可以正常使用：                                          │
│     - 登录表单功能正确                                                │
│     - 无 reCAPTCHA（默认不启用）                                      │
│     - 主题是正确的 flatly                                             │
└──────────────────────────────────────────────────────────────────────┘
         │
         │ 用户输入密码后点击 "Go"
         ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 4. POST /login（正常表单提交）                                        │
│                                                                      │
│  Login::index()                                                      │
│    ├─ APP_STATE.isNewInstall = false → JS 不拦截 → 正常提交          │
│    ├─ is_latest() = true → 不执行迁移                                │
│    ├─ config(OSPOS::class)->settings ← 新 OSPOS 实例                 │
│    │   └─ set_settings()                                             │
│    │       ├─ cache->get('settings') → false（从未写入缓存）         │
│    │       ├─ app_config 表已存在 → 查询数据库                       │
│    │       ├─ 读取所有配置 → $this->settings                         │
│    │       └─ cache->save('settings', ...) → 写入缓存               │
│    ├─ 验证用户名密码                                                  │
│    └─ 成功 → redirect()->to('home')                                  │
│                                                                      │
│  ✅ 配置完全是最新的：缓存从未写入过旧值 → 必定从数据库读取            │
└──────────────────────────────────────────────────────────────────────┘
         │
         │ 302 重定向
         ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 5. GET /home                                                         │
│                                                                      │
│  Home 控制器 → 新 OSPOS 实例                                         │
│    ├─ cache->get('settings') → 命中（步骤 4 已写入）                 │
│    └─ 使用完整配置渲染页面 ✅                                         │
└──────────────────────────────────────────────────────────────────────┘
```

### 8.8 失败后重试的完整流程

```
┌──────────────────────────────────────────────────────────────────────┐
│ AJAX POST /migrate → 失败                                            │
│                                                                      │
│  Login::migrate()                                                    │
│    ├─ latest() 执行中                                                │
│    │   ├─ 迁移 A: up() 成功 → addHistory(batch=2) ✓                 │
│    │   └─ 迁移 B: up() 抛异常 ✗                                      │
│    │       ├─ regress(-1) 自动触发                                   │
│    │       ├─ A.down() → removeHistory(A) → A 的历史记录删除         │
│    │       └─ 抛出 RuntimeException                                  │
│    └─ catch → return JSON {success: false, message: "..."} → 500    │
│                                                                      │
│  数据库状态：                                                         │
│    ├─ migrations 表: batch=2 记录被 regress 清除                     │
│    ├─ app_config 表: A 插入的配置可能仍在（down() 为空时）            │
│    └─ 缓存：从未写入（全新安装场景）                                  │
│                                                                      │
│  前端状态：                                                           │
│    ├─ APP_STATE.isNewInstall = true（未被修改）                       │
│    └─ showMigrationError() → 显示错误 + "迁移"按钮可再次点击         │
└──────────────────────────────────────────────────────────────────────┘
         │
         │ 用户再次点击 "迁移"
         ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 第二次 AJAX POST /migrate → 重试                                     │
│                                                                      │
│  Login::migrate()                                                    │
│    ├─ new MY_Migration()                                             │
│    ├─ migrate_to_ci4() → 已完成，跳过                                │
│    ├─ latest()                                                       │
│    │   ├─ findMigrations() → 所有迁移文件                             │
│    │   ├─ getHistory() → 不含 A、B（已被 regress 清除）              │
│    │   ├─ getLastBatch() = 1 → 新 batch=2                            │
│    │   ├─ 迁移 A: up() 重新执行                                      │
│    │   │   └─ INSERT IGNORE → 配置已存在，跳过 → 成功 ✅             │
│    │   │   └─ addHistory(batch=2)                                    │
│    │   ├─ 迁移 B: up() 重新执行                                      │
│    │   │   └─ （期望这次成功）                                        │
│    │   │   └─ addHistory(batch=2)                                    │
│    │   └─ ... 后续迁移 ...                                           │
│    └─ return JSON {success: true} ✅                                 │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 九、关键风险点与注意事项

### 8.1 迁移失败的常见原因

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

### 8.2 配置补齐的边界情况

1. **全新安装**：`initial_schema.sql` 一次性插入所有基础配置，后续迁移只补新增项

2. **跨版本升级**：如从 3.2.0 直接升级到 3.4.2
   - 各版本迁移按顺序执行
   - 每个版本的配置补齐只补该版本新增的
   - `INSERT IGNORE` 保证即使前面版本漏了也不会冲突

3. **用户手动删除配置**：
   - 重新执行迁移不会恢复（已存在于 migrations 历史）
   - 需要手动插入或使用 `Appconfig::save()`

### 8.3 迁移后缓存未刷新的风险

**关键问题**：通过 SQL `INSERT IGNORE` 补齐的配置，在迁移完成后可能不会立即生效。

**原因链**：
1. 迁移执行前 OSPOS 实例已加载旧配置到内存
2. SQL 直接插入数据库，不经过 `Appconfig::save()`
3. 缓存未被删除，仍保留旧数据
4. 新请求如果缓存未过期（5分钟内），仍然读到旧缓存

**改进建议**：在迁移全部成功后主动刷新缓存：
```php
// 在 Login::index() 中 latest() 成功后添加
config(OSPOS::class)->update_settings();
```

### 8.4 性能考量

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

## 九、配置生效时机总结

### 9.1 配置生效的完整时间线

```
T0: 代码升级完成
    ↓
T1: 用户首次访问 /login
    ├─ Login::index() 实例化
    ├─ OSPOS 实例化 → set_settings() → 加载旧配置（内存+缓存）
    ├─ is_latest() = false → 显示迁移提示
    └─ post_controller_constructor 事件
        └─ Load_config::load_config()
            ├─ is_latest() = false → session->destroy()
            └─ 使用旧配置设置语言/时区
    ↓
T2: 用户点击迁移按钮
    ├─ AJAX POST /migrate
    ├─ 新请求 → 新 OSPOS 实例 → 加载旧配置
    ├─ latest() 开始执行
    │   ├─ 迁移 A: SQL INSERT IGNORE 配置 X
    │   │   ├─ 数据库：X 已写入 ✅
    │   │   ├─ 缓存：未变 ❌
    │   │   └─ 内存：无 X ❌
    │   ├─ 迁移 B: $appconfig->save(['Y' => 'val'])
    │   │   ├─ 数据库：Y 已写入 ✅
    │   │   ├─ 触发 update_settings()
    │   │   │   ├─ cache->delete('settings') ✅
    │   │   │   └─ set_settings() 重新加载
    │   │   │       ├─ 缓存已删 → 读数据库
    │   │   │       └─ 内存：有 X 和 Y ✅
    │   │   └─ 缓存：已删除，下次读库重建
    │   └─ ... 更多迁移 ...
    ├─ 迁移全部成功
    └─ 返回 JSON {success: true}
    ↓
T3: 前端检测成功，刷新页面
    ├─ GET /login → 新请求
    ├─ 新 OSPOS 实例化 → set_settings()
    │   ├─ 检查缓存：
    │   │   ├─ 场景 A：迁移 B 刷新过缓存 → 缓存已删 → 读数据库 → 最新配置 ✅
    │   │   └─ 场景 B：所有迁移都是 SQL 直接插入 → 缓存可能还在（<5分钟）
    │   │       ├─ 缓存命中 → 旧配置 → 无新配置 ❌
    │   │       └─ 缓存过期 → 读数据库 → 最新配置 ✅
    ├─ is_latest() = true → 显示登录表单
    └─ post_controller_constructor 事件
        └─ Load_config::load_config()
            ├─ is_latest() = true → 不销毁 session
            └─ 使用当前配置（可能是旧缓存）设置语言/时区
```

### 10.2 确保配置立即生效的方法

1. **迁移中使用 `Appconfig::save()`** 而非直接 SQL：
   ```php
   // 推荐：自动刷新缓存
   $appconfig = model(Appconfig::class);
   $appconfig->save(['new_key' => 'value']);
   
   // 避免：需要手动刷新
   $this->db->table('app_config')->ignore(true)->insertBatch($values);
   ```

2. **迁移完成后主动刷新缓存**：
   ```php
   // 在 Login::index() 中 latest() 成功后
   $migration->latest();
   config(OSPOS::class)->update_settings();  // 主动刷新
   ```

3. **手动清除缓存文件**：
   ```bash
   rm -rf writable/cache/*
   ```

---

## 十、代码路径索引

### 版本管理
- 版本读取：[MY_Migration::get_current_version()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Libraries/MY_Migration.php#L36-L53)
- 最新版本：[MY_Migration::get_latest_migration()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Libraries/MY_Migration.php#L25-L29)
- 版本比较：[MY_Migration::is_latest()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Libraries/MY_Migration.php#L14-L20)

### 迁移执行
- 入口检测：[Login::index()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Controllers/Login.php#L23-L57)
- AJAX 接口：[Login::migrate()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Controllers/Login.php#L77-L99)
- CI3→CI4 转换：[MY_Migration::migrate_to_ci4()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Libraries/MY_Migration.php#L58-L64)
- SQL 执行：[migration_helper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Helpers/migration_helper.php)

### 配置加载与缓存
- 配置实例化：[OSPOS::__construct()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Config/OSPOS.php#L21-L26)
- 配置加载：[OSPOS::set_settings()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Config/OSPOS.php#L31-L56)
- 缓存刷新：[OSPOS::update_settings()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Config/OSPOS.php#L71-L75)
- 默认值兜底：[OSPOS::getDefaultSettings()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Config/OSPOS.php#L58-L66)
- 自动刷新触发：[Appconfig::save()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Models/Appconfig.php#L77-L90)
- 缓存配置：[Cache.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Config/Cache.php#L25-L87)
- 编码/解码：[encode_array()/decode_array()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Helpers/locale_helper.php#L665-L685)
- 事件触发：[Load_config::load_config()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Events/Load_config.php#L24-L45)
- 事件注册：[Events.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Config/Events.php#L61)

### 中断恢复
- 批次回滚：CI4 `MigrationRunner::regress()`
- 幂等插入：`$builder->ignore(true)->insertBatch()`
- 存在性检查：[migration_helper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Helpers/migration_helper.php#L223-L237) 中 `indexExists()`, `foreignKeyExists()`, `primaryKeyExists()`
- 迁移执行时序：`MigrationRunner::latest()` 中 `migrate('up')` → `addHistory()` → 失败 `regress(-1)`

### 配置补齐
- 初始配置：[initial_schema.sql](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/sqlscripts/initial_schema.sql#L6-L85)
- 配置模型：[Appconfig.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Models/Appconfig.php)
- 配置迁移示例：
  - PHP 方式：[20200508000000_image_upload_defaults.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/20200508000000_image_upload_defaults.php)
  - PHP 方式：[20230412000000_add_missing_config.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/20230412000000_add_missing_config.php)
  - PHP 方式：[20260506000000_AddShortcutKeys.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/20260506000000_AddShortcutKeys.php)
  - SQL 方式：[3.4.2_missing_config_keys.sql](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/sqlscripts/3.4.2_missing_config_keys.sql)
  - 模型方式：[20240319000000_Migration_Convert_Barcode_Types.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/20240319000000_Migration_Convert_Barcode_Types.php)

### 回滚边界分析
- 空 down() 示例：[20250716170000_MissingConfigKeys.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/20250716170000_MissingConfigKeys.php#L21-L24)
- 不删除配置示例：[20230412000000_add_missing_config.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/20230412000000_add_missing_config.php#L32-L35)
- 删除配置示例：[20200508000000_image_upload_defaults.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/20200508000000_image_upload_defaults.php#L29-L34)
- 不可逆示例：[20250521000000_FixImageFilenameSpaces.php](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Database/Migrations/20250521000000_FixImageFilenameSpaces.php#L60-L64)
- CI4 转换：[MY_Migration::migrate_table()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Libraries/MY_Migration.php#L84-L110)

### AJAX 迁移前端代码路径
- JS 状态管理：[login.php APP_STATE](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Views/login.php#L170-L189)
- 表单提交拦截：[login.php $form.on('submit')](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Views/login.php#L257-L288)
- 成功处理：[login.php showMigrationSuccess()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Views/login.php#L221-L230)
- 失败处理：[login.php showMigrationError()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Views/login.php#L232-L241)
- 登录表单显示：[login.php showLoginForm()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Views/login.php#L243-L251)
- 路径 A 入口：[Login::index() POST 分支](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Controllers/Login.php#L48-L57)
- 路径 B 入口：[Login::migrate()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Controllers/Login.php#L77-L99)
- SQL 非事务执行：[execute_script()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Helpers/migration_helper.php#L10-L42)
- SQL 事务执行：[executeScriptWithTransaction()](file:///d:/fz/0601-1/solo-dogfeeding/code/20-opensourcepos/app/Helpers/migration_helper.php#L49-L88)
