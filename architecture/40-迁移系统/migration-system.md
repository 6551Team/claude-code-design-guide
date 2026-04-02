# 配置迁移系统

> 所有迁移在应用启动时自动运行, 采用幂等设计 -- 重复执行不会产生副作用。迁移系统确保用户配置在版本升级时平滑过渡。

---

## 概述

### 设计原则

- **启动时执行**: 所有迁移在应用初始化阶段顺序运行
- **幂等性**: 每个迁移函数可安全重复执行, 已完成的迁移不会重复应用
- **向前兼容**: 迁移只做单向转换, 不支持回滚
- **无损转换**: 迁移过程中不丢失用户数据

### 设计理念：为什么幂等设计？

用户可能在迁移中途意外退出（崩溃、强制关闭、断电），下次启动时迁移会再次执行。源码中的幂等守卫模式（如 `migrateEnableAllProjectMcpServersToSettings.ts` 第 54-56 行的 `"Already migrated, just mark for removal"` 注释）确保：
- 已完成的迁移被安全跳过
- 部分完成的迁移可以从断点继续而非产生不一致状态

### 设计理念：为什么只支持前向迁移不支持回滚？

- **降级场景极少** -- 用户几乎不会从新版本回退到旧版本
- **回滚逻辑倍增复杂度** -- 每个迁移都需要维护正向和反向两套逻辑，任何一个回滚实现的 bug 都可能导致数据丢失
- **配置变更通常不可逆** -- 例如模型名称从 `fennec` 迁移到 `opus` 后，旧名称已失效，回滚没有意义

### 设计理念：为什么在启动时同步执行？

迁移影响配置读取——如果异步执行，可能出现竞态条件（应用读到未迁移的旧配置）。源码中迁移在 `应用启动 → 遍历 migrationsList[] → 应用正常启动` 的同步流程中顺序执行，确保：
- 所有后续代码读取到的配置都是已迁移的最新版本
- 不存在"配置读取"与"配置迁移"之间的竞态窗口

### 执行流程

![迁移系统执行流程](../diagrams/migration-execution-flow.svg)

---

## 迁移清单

| #  | 函数名                                            | 说明                          | 迁移方向                              |
|----|--------------------------------------------------|-------------------------------|---------------------------------------|
| 1  | `migrateAutoUpdatesToSettings`                   | 功能标志迁移至 settings        | Feature Flag -> `settings.json`       |
| 2  | `migrateBypassPermissionsAcceptedToSettings`     | 权限接受配置迁移至 settings     | 权限配置 -> `settings.json`            |
| 3  | `migrateEnableAllProjectMcpServersToSettings`    | MCP 启用配置迁移至 settings    | MCP 配置 -> `settings.json`           |
| 4  | `migrateFennecToOpus`                            | Fennec 模型名更新为 Opus       | `fennec` -> `opus`                    |
| 5  | `migrateLegacyOpusToCurrent`                     | 旧版 Opus 标识更新             | 旧 Opus ID -> 当前 Opus ID            |
| 6  | `migrateOpusToOpus1m`                            | Opus 升级为 Opus 1M 上下文     | `opus` -> `opus-1m`                   |
| 7  | `migrateReplBridgeEnabledToRemoteControlAtStartup` | REPL Bridge 迁移至远程控制  | REPL Bridge -> Remote Control         |
| 8  | `migrateSonnet1mToSonnet45`                      | Sonnet 1M 迁移至 Sonnet 4.5   | `sonnet-1m` -> `sonnet-4.5`           |
| 9  | `migrateSonnet45ToSonnet46`                      | Sonnet 4.5 迁移至 Sonnet 4.6  | `sonnet-4.5` -> `sonnet-4.6`          |
| 10 | `resetAutoModeOptInForDefaultOffer`              | 自动模式选择重置               | 重置 opt-in 状态为默认                  |
| 11 | `resetProToOpusDefault`                          | Pro 用户重置为 Opus 默认       | Pro 模型偏好 -> Opus 默认               |

---

## 迁移分类

### 设置迁移 (Settings Migrations)

将分散的功能标志和配置项统一迁移到 `settings.json`:

```typescript
// 迁移 #1: Feature Flag -> settings
function migrateAutoUpdatesToSettings(): void

// 迁移 #2: 权限配置 -> settings
function migrateBypassPermissionsAcceptedToSettings(): void

// 迁移 #3: MCP 配置 -> settings
function migrateEnableAllProjectMcpServersToSettings(): void
```

### 模型名称迁移 (Model Name Migrations)

随着模型版本演进, 自动更新用户配置中的模型标识:

![模型名称迁移路径](../diagrams/migration-model-name-paths.svg)

### 功能迁移 (Feature Migrations)

```typescript
// 迁移 #7: REPL Bridge -> Remote Control
function migrateReplBridgeEnabledToRemoteControlAtStartup(): void
// REPL Bridge 功能已被远程控制取代
// 将旧配置映射到新的远程控制选项
```

### 重置迁移 (Reset Migrations)

```typescript
// 迁移 #10: 当默认 offer 变更时, 重置用户的 opt-in 状态
function resetAutoModeOptInForDefaultOffer(): void

// 迁移 #11: Pro 订阅用户的默认模型重置为 Opus
function resetProToOpusDefault(): void
```

---

## 幂等性保障

每个迁移遵循统一的幂等模式:

```typescript
function migrationTemplate(): void {
  // 1. 读取当前配置状态
  const currentValue = readConfig('some.key');

  // 2. 检查是否需要迁移 (幂等守卫)
  if (currentValue === undefined || isAlreadyMigrated(currentValue)) {
    return; // 已完成或不适用, 跳过
  }

  // 3. 执行迁移
  writeConfig('new.key', transformValue(currentValue));

  // 4. 清理旧配置 (可选)
  removeConfig('some.key');
}
```

---

## 添加新迁移

新增迁移时需注意:

1. 在 `migrationsList` 数组末尾追加 (保持顺序)
2. 确保函数是幂等的 -- 多次调用结果一致
3. 处理配置不存在的情况 (新安装用户)
4. 添加对应的单元测试

### 工程实践

**添加新迁移的完整清单**：

1. 在 `migrationsList` 数组**末尾**追加新迁移函数（绝不能插入中间位置，保持执行顺序）
2. 迁移函数必须是幂等的——多次调用产生完全相同的结果，参考源码中的统一模式：先读取 → 检查是否需要迁移（幂等守卫）→ 执行 → 清理
3. 处理配置不存在的情况——新安装用户什么都没有，`readConfig()` 可能返回 `undefined`
4. 添加单元测试覆盖三种情况：
   - 已迁移状态（应跳过，不产生副作用）
   - 未迁移状态（应正确执行迁移）
   - 配置不存在状态（应安全处理，不抛异常）

**迁移测试模式**：
- 可以通过直接调用迁移函数 + mock 配置文件来测试，无需启动完整应用
- 利用幂等性，可以在测试中连续调用两次迁移函数，验证第二次调用不产生变更

**常见陷阱**：
- **不要依赖其他迁移的结果** -- 每个迁移必须独立工作，不能假设前面的迁移已执行。虽然 `migrationsList` 按顺序执行，但如果迁移 A 依赖迁移 B 的结果，当 B 失败时 A 也会级联失败
- **不要在迁移中做网络调用** -- 迁移在启动时同步执行，网络调用会阻塞启动流程，且离线场景下会导致应用无法启动
- **不要删除旧配置字段的代码** -- 保留旧字段的读取逻辑作为兼容层，直到确认所有用户都已迁移


---

[← 原生模块](../39-原生模块/native-modules.md) | [目录](../README.md) | [文件持久化 →](../41-文件持久化/file-persistence.md)
