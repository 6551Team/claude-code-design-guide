# 沙箱系统架构文档

> Claude Code v2.1.88 沙箱安全隔离系统完整技术参考

---

### 设计理念

#### 为什么沙箱是可选的？

不是所有环境都支持沙箱——Docker 容器内无法嵌套 Linux namespace，某些 CI/CD 环境没有 sandbox-exec 权限，远程开发环境可能有自己的隔离层。`failIfUnavailable` 配置项（`sandboxTypes.ts`）让用户选择行为：`true` 时沙箱不可用则报错终止（适合安全要求高的场景），`false` 时降级为无沙箱运行（适合功能优先的场景）。`enableWeakerNestedSandbox` 和 `enableWeakerNetworkIsolation` 提供了弱化的隔离选项作为兼容性折中。

#### 为什么 3 种沙箱类型？

每种 OS 有不同的原生隔离机制——macOS 使用 `sandbox-exec`（Seatbelt），Linux 使用 user namespace + cgroup，Docker 环境使用容器隔离。统一为一种方案不可行，因为底层系统调用完全不同。沙箱适配层（`sandbox-adapter.ts`）在初始化时检测环境并选择合适的实现，对上层提供统一接口。

#### 为什么允许粒度权限配置？

不同工具需要不同权限——Bash 可能需要网络访问来 `npm install`，FileRead 只需要文件系统读权限，`excludedCommands` 列表允许特定命令绕过沙箱。`autoAllowBashIfSandboxed` 的存在说明了一种务实的权衡：当沙箱已经提供了隔离保护时，可以减少用户确认提示，改善交互流畅度。

## 沙箱配置 (settings.json sandbox字段)

```typescript
sandbox: {
  enabled: boolean,                      // 启用沙箱
  failIfUnavailable: boolean,            // 不可用时失败 vs 降级运行
  allowUnsandboxedCommands: boolean,      // 允许非沙箱命令执行
  network: {...},                         // 网络限制配置
  filesystem: {...},                      // 文件系统限制配置
  ignoreViolations: boolean,             // 忽略违规（不阻止执行）
  excludedCommands: string[],            // 排除的命令（不经过沙箱）
  autoAllowBashIfSandboxed: boolean,     // 沙箱模式下自动允许bash
  enableWeakerNestedSandbox: boolean,    // 启用弱嵌套沙箱
  enableWeakerNetworkIsolation: boolean, // 启用弱网络隔离
  ripgrep: {...}                         // ripgrep特定配置
}
```

### 配置字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `enabled` | boolean | 是否启用沙箱隔离 |
| `failIfUnavailable` | boolean | 沙箱不可用时的行为：`true` 报错终止，`false` 降级为无沙箱运行 |
| `allowUnsandboxedCommands` | boolean | 是否允许执行未沙箱化的命令 |
| `ignoreViolations` | boolean | 是否忽略沙箱违规报告 |
| `excludedCommands` | string[] | 不经过沙箱的命令白名单 |
| `autoAllowBashIfSandboxed` | boolean | 沙箱模式下自动批准 bash 命令（无需用户确认） |
| `enableWeakerNestedSandbox` | boolean | 允许使用弱化的嵌套沙箱（兼容性选项） |
| `enableWeakerNetworkIsolation` | boolean | 使用弱化的网络隔离策略 |

---

## 沙箱执行 (sandbox-adapter.ts)

### 初始化
```
M7.initialize(SK8)  // 异步初始化沙箱引擎
```

### 不可用处理
- `failIfUnavailable = true` → 报错，阻止执行
- `failIfUnavailable = false` → 降级为无沙箱运行

### 命令执行决策
```
shouldUseSandbox()  // 决定当前命令是否使用沙箱
```
考虑因素：沙箱可用性、命令排除列表、`dangerouslyDisableSandbox` 参数等。

### BashTool 集成
`BashTool` 的 `dangerouslyDisableSandbox` 参数可显式绕过沙箱保护（需要权限授权）。

---

## 违规检测

### removeSandboxViolationTags(text)
从错误消息中移除 `<sandbox_violations>` 标签，清理内部标记后再展示给用户。

### 违规处理流程
1. 沙箱检测到违规行为
2. 违规消息格式化
3. 根据 `ignoreViolations` 设置决定是否阻止
4. 显示给用户（如果不忽略）

---

## 网络控制

### MITM 代理
- 使用中间人代理拦截网络请求
- 被阻止的请求返回: `X-Proxy-Error: blocked-by-allowlist`
- 支持域名白名单和黑名单机制

### 代理套接字
```
getMitmSocketPath()  // 获取代理套接字路径
```

### 上游代理 (src/upstreamproxy/)

#### relay.ts (456行)
TCP 到 WebSocket 到 CCR 的隧道中继。

**关键实现细节**:

| 特性 | 说明 |
|------|------|
| Protobuf 编码 | 手写 varint 编码/解码（不依赖外部 protobuf 库） |
| 背压处理 | Bun 部分写入 vs Node 缓冲的差异处理 |
| Keepalive | 30秒间隔的 pinger 保持连接活跃 |

---

## Swarm 中的沙箱权限

在多智能体 Swarm 模式下，沙箱权限通过邮箱系统在工作者和领导者之间传递。

### 权限请求
```
sendSandboxPermissionRequestViaMailbox()
```
工作者 → 领导者：发送沙箱权限请求。

### 权限响应
```
sendSandboxPermissionResponseViaMailbox()
```
领导者 → 工作者：返回沙箱权限决策结果。

### 流程
1. 工作者智能体需要执行受限操作
2. 工作者通过邮箱发送权限请求给领导者
3. 领导者评估请求并做出决策
4. 领导者通过邮箱返回允许/拒绝响应
5. 工作者根据响应继续或中止操作

---

## 工程实践指南

### 启用/禁用沙箱

**配置 `settings.json` 中的 `sandbox` 字段：**

```json
{
  "sandbox": {
    "enabled": true,
    "failIfUnavailable": false,
    "autoAllowBashIfSandboxed": true,
    "excludedCommands": ["specific-command"],
    "enableWeakerNestedSandbox": false,
    "enableWeakerNetworkIsolation": false
  }
}
```

**沙箱类型选择（自动检测）：**

| 环境 | 沙箱实现 | 说明 |
|------|---------|------|
| macOS | `sandbox-exec`（Seatbelt） | macOS 原生沙箱 |
| Linux | user namespace + cgroup | Linux 原生隔离 |
| Docker | 容器隔离 | 使用宿主容器的隔离层 |
| Windows | 无原生沙箱 | `sandbox-exec`/`bwrap` 不可用 |

**failIfUnavailable 行为：**
- `true` → 沙箱不可用时报错终止（安全要求高的场景）
- `false` → 降级为无沙箱运行（功能优先的场景）

### 调试沙箱违规

**排查步骤：**

1. **检查违规消息**：沙箱检测到违规行为后，会格式化违规消息并根据 `ignoreViolations` 决定是否阻止
2. **查看违规标签**：`removeSandboxViolationTags(text)` 从错误消息中移除 `<sandbox_violations>` 标签——内部标记清理后再展示给用户
3. **检查命令排除列表**：`excludedCommands` 中的命令不经过沙箱
4. **检查 `dangerouslyDisableSandbox` 参数**：BashTool 支持此参数显式绕过沙箱（需权限授权）
   - 源码 `BashTool/prompt.ts` 指示：默认在沙箱中运行，仅在命令因沙箱限制失败后才用 `dangerouslyDisableSandbox: true` 重试
   - 当 `allowUnsandboxedCommands = false` 时，此参数被完全忽略

**`shouldUseSandbox()` 决策因素：**
- 沙箱可用性
- 命令排除列表
- `dangerouslyDisableSandbox` 参数
- 沙箱配置中的各项开关

### 自定义沙箱规则

**文件系统规则：**
- 通过 `sandbox.filesystem` 配置允许/禁止的目录
- 工作目录和项目目录通常在允许列表中

**网络规则：**
- `sandbox.network` 配置网络访问权限
- MITM 代理拦截网络请求，被阻止的请求返回 `X-Proxy-Error: blocked-by-allowlist`
- `enableWeakerNetworkIsolation` 提供弱化的网络隔离（兼容性折中）
- 上游代理（`relay.ts`）使用 TCP→WebSocket→CCR 隧道中继，手写 varint 编码，30 秒 keepalive

**弱化选项（兼容性折中）：**
- `enableWeakerNestedSandbox` — 在已有隔离层的环境中使用弱化嵌套沙箱
- `enableWeakerNetworkIsolation` — 使用弱化网络隔离策略

### Swarm 模式下的沙箱权限

**在多智能体模式下：**
1. 工作者通过 `sendSandboxPermissionRequestViaMailbox()` 请求权限
2. 领导者通过 `sendSandboxPermissionResponseViaMailbox()` 返回决策
3. 权限通过邮箱系统传递，确保跨智能体的安全决策一致

### 常见陷阱

| 陷阱 | 详情 | 解决方案 |
|------|------|----------|
| Docker 内无法嵌套 namespace 沙箱 | Linux namespace 在 Docker 容器内不可用 | 使用 `enableWeakerNestedSandbox` 或 `failIfUnavailable: false` 降级 |
| 沙箱可能阻止正常工具操作 | `npm install`、`pip install` 等需要网络的命令可能被拦截 | 配置 `excludedCommands` 或允许网络访问 |
| `autoAllowBashIfSandboxed` 的安全权衡 | 沙箱已提供隔离时减少用户确认提示 | 适合开发环境但生产/安全敏感环境应谨慎 |
| Windows 无原生沙箱 | `sandbox-exec`/`bwrap` 在 Windows 上不可用 | 源码注释确认 PowerShell tool 在 Windows native 上 sandbox 不可用 |
| 沙箱违规可能被忽略 | `ignoreViolations: true` 时违规不阻止执行 | 仅在调试/开发场景使用此选项 |
| 每个命令独立评估沙箱 | 即使最近用了 `dangerouslyDisableSandbox`，后续命令仍默认沙箱运行 | 源码 prompt 明确要求"Treat each command individually" |


---

[← LSP 集成](../23-LSP集成/lsp-integration.md) | [目录](../README.md) | [Git 与 GitHub →](../25-Git与GitHub/git-github.md)
