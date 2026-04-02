# OAuth 与认证系统

Claude Code 的认证系统实现了完整的 OAuth 2.0 PKCE 流程、安全令牌存储和多源认证解析链。

---

## OAuth Service (src/services/oauth/)

### 设计理念

#### 为什么用 PKCE 而不是简单的 OAuth？

Claude Code 是桌面 CLI 应用，属于 OAuth 规范中的"公共客户端"——无法安全存储 client_secret（任何嵌入在本地二进制中的 secret 都可以被逆向提取）。PKCE（Proof Key for Code Exchange）专为这种场景设计：用一次性的 `code_verifier`/`code_challenge` 对替代 client_secret，即使授权码被截获也无法交换 token。源码 `OAuthService` 构造函数中 `this.codeVerifier = crypto.generateCodeVerifier()` 每次流程生成唯一的验证器，`client.ts` 中 `code_challenge_method=S256` 使用 SHA-256 哈希，确保 verifier 不以明文传输。

### OAuthService 类

核心 OAuth 服务类，管理完整的认证流程。

#### 主要方法

**startOAuthFlow()**
- 启动完整的 OAuth 认证流程
- 流程：localhost listener → PKCE + state → build URLs → wait for code → exchange → profile fetch
- 自动打开浏览器跳转到授权页面
- 在本地启动 HTTP 服务器监听回调

**handleManualAuthCodeInput()**
- 当自动流程失败时（如浏览器未打开），支持用户手动粘贴授权码
- 解析用户输入的授权码并继续 token exchange 流程

**cleanup()**
- 清理 OAuth 流程中的临时资源
- 关闭 localhost HTTP listener
- 清除临时状态

### OAuth 流程详细步骤

1. 生成 PKCE code_verifier 和 code_challenge
2. 生成随机 state 参数防止 CSRF
3. 启动本地 HTTP 服务器监听回调
4. 构建授权 URL 并打开浏览器
5. 等待用户授权后的回调（携带 authorization code）
6. 使用 code + code_verifier 交换 tokens
7. 获取用户 profile 信息

---

## Token Exchange (client.ts)

### buildAuthUrl

```typescript
buildAuthUrl(codeChallenge: string, state: string) → string
```

构建 OAuth 授权 URL，包含：
- client_id
- redirect_uri (localhost)
- response_type=code
- code_challenge + code_challenge_method=S256
- state
- scope

### exchangeCodeForTokens

```typescript
exchangeCodeForTokens() → { accessToken, refreshToken, expiresIn, scope }
```

使用 authorization code 交换 access token 和 refresh token：
- 发送 POST 请求到 token endpoint
- 包含 code_verifier 用于 PKCE 验证
- 内置重试机制处理临时网络错误

### refreshOAuthToken

```typescript
refreshOAuthToken(refreshToken: string, scopes?: string[]) → { accessToken, refreshToken, expiresIn }
```

使用 refresh token 刷新 access token：
- 支持指定 scope 缩小权限范围
- 返回新的 token 对（access + refresh）

#### 为什么自动 token 刷新？

用户不应该因为 token 过期而中断工作流——无感刷新是最佳 UX。`isOAuthTokenExpired()` 内置 5 分钟缓冲期（在实际过期前就返回 `true`），确保有足够时间完成刷新。`jwtUtils.ts` 中的 `createTokenRefreshScheduler` 甚至实现了主动刷新——根据 token 的过期时间提前调度刷新任务，而非等到请求失败时才被动刷新。当刷新失败时，系统才回退到重新登录流程，最大程度减少对用户工作的打断。

### fetchProfileInfo

```typescript
fetchProfileInfo(accessToken: string) → {
  subscription: string,
  rateLimitTier: string,
  displayName: string,
  billingType: string,
}
```

获取用户 profile 信息：
- subscription: 订阅类型（free, pro, max 等）
- rateLimitTier: 速率限制层级
- displayName: 用户显示名称
- billingType: 计费类型

### isOAuthTokenExpired

```typescript
isOAuthTokenExpired() → boolean
```

检查当前 OAuth token 是否过期：
- 内置 5 分钟缓冲期，在实际过期前就返回 true
- 确保有足够时间完成 token 刷新

### populateOAuthAccountInfoIfNeeded

```typescript
populateOAuthAccountInfoIfNeeded() → void
```

惰性获取 OAuth 账户信息：
- 仅在需要时（如首次访问账户信息）才发起请求
- 获取后缓存结果，避免重复请求

---

## Auth Code Listener (auth-code-listener.ts)

### AuthCodeListener 类

在本地启动 HTTP 服务器，接收 OAuth 回调。

#### start

```typescript
start(port?: number) → Promise<number>
```

- 启动 localhost HTTP 服务器
- 如未指定端口，使用 OS 分配的随机端口（port 0）
- 返回实际监听的端口号

#### waitForAuthorization

```typescript
waitForAuthorization(state: string, onReady: (url: string) => void) → Promise<string>
```

- 等待 OAuth provider 的回调请求
- `state` 参数用于验证回调的合法性
- `onReady` 回调在服务器就绪时触发，传入重定向 URL

#### 安全验证与错误处理

- **State mismatch**: 回调中的 state 与预期不匹配 → reject，防止 CSRF 攻击
- **Missing code**: 回调中缺少 authorization code → 返回 HTTP 400
- **Success redirect**: 认证成功后重定向到成功页面
- **Error redirect**: 认证失败后重定向到错误页面，显示错误信息

---

## Secure Storage (src/utils/secureStorage/)

### 设计理念

#### 为什么 keychain 集成？

明文存储 token 是安全反模式——任何能读取用户文件的进程都能窃取凭证。OS keychain（macOS Keychain / Windows Credential Manager）提供了操作系统级的加密存储，凭证受到用户登录密码保护。macOS 实现（`macOsKeychainStorage.ts`）通过 `security` CLI 工具与 Keychain 交互，优先使用 stdin 传递凭证（比命令行参数更安全，不暴露在进程列表中），当凭证超过 4032 bytes（`SECURITY_STDIN_LINE_LIMIT`）时才降级为 argv 传参。凭证值通过 hex 编码存储，规避进程监控工具对命令行中明文的捕获。5 分钟的 `KEYCHAIN_CACHE_TTL_MS` 减少对系统 Keychain 的频繁访问，stale-while-error 策略在 token 刷新失败时使用缓存旧值继续工作——这些都是为了在安全性和可用性之间取得平衡。

### 平台抽象

根据运行平台选择不同的存储后端：

- **macOS**: keychain 存储 + plaintext 降级方案
- **Windows/Linux**: plaintext 存储

### plainTextStorage

```
存储位置: ~/.claude/.credentials.json
文件权限: 0600 (仅所有者可读写)
```

- 将凭证以 JSON 格式存储在文件系统
- 通过严格的文件权限保护安全性
- 作为所有平台的降级存储方案

### macOsKeychainStorage

使用 macOS 系统 Keychain 进行安全存储：

```
命令: security find-generic-password / security add-generic-password
```

#### KEYCHAIN_CACHE_TTL_MS

```typescript
const KEYCHAIN_CACHE_TTL_MS = 5 * 60 * 1000  // 5 分钟
```

Keychain 读取结果的内存缓存 TTL，减少对系统 Keychain 的频繁访问。

#### Stale-while-error 策略

当 token 刷新失败时，如果缓存中有旧值（即使已过 TTL），仍使用旧值继续工作，而非立即失败。类似 HTTP 的 stale-while-revalidate 模式。

#### Hex 编码

```
凭证值通过 hex 编码后再存入 Keychain
```

使用十六进制编码存储凭证内容，规避进程监控工具对命令行参数中明文凭证的捕获。

#### stdin 限制与降级

```
stdin 传输限制: 4032 bytes
超过限制: 降级为 argv 传参
```

- 优先通过 stdin 传递凭证给 `security` 命令（更安全，不暴露在进程参数中）
- 当凭证超过 4032 bytes 时，降级为通过命令行参数传递

#### isMacOsKeychainLocked

```typescript
isMacOsKeychainLocked() → boolean
```

检查 macOS Keychain 是否处于锁定状态：
- 结果会被缓存，避免频繁检测
- Keychain 锁定时自动降级到 plaintext 存储

---

## Auth Resolution Chain

### 认证源优先级

认证系统按以下优先级依次尝试，首个成功的源被使用：

```
1. 3P context        → 第三方集成上下文提供的凭证
2. bare mode         → bare 模式下的特殊认证
3. managed OAuth     → 托管环境的 OAuth 凭证
4. explicit tokens   → 显式配置的 token
5. OAuth             → 标准 OAuth 流程获取的 token
6. API key fallback  → API key 降级
```

### 认证源详细说明

**3P Context (第三方上下文)**
- 当 Claude Code 作为第三方工具集成时，由宿主环境提供凭证
- 最高优先级，直接使用

**Bare Mode**
- 精简模式下的认证，绕过标准流程

**Managed OAuth**
- 企业或托管环境预配置的 OAuth 凭证

**Explicit Tokens**
- `ANTHROPIC_AUTH_TOKEN` 环境变量
- `apiKeyHelper` 外部程序获取 token
- `FILE_DESCRIPTOR` 通过文件描述符传递 token

**OAuth (claude.ai)**
- 标准 OAuth 2.0 PKCE 流程
- 通过 claude.ai 授权获取 token
- 支持自动刷新

**API Key Fallback**
- `ANTHROPIC_API_KEY` 环境变量
- 最低优先级的降级方案
- 直接使用 API key 而非 OAuth token

---

## 工程实践指南

### 调试认证失败

**排查步骤清单：**

1. **检查 keychain 中的 token**：
   - macOS：`security find-generic-password -s claude-code` 查看 keychain 条目
   - Windows/Linux：检查 `~/.claude/.credentials.json`（权限 0600）
2. **手动刷新 token**：
   - `isOAuthTokenExpired()` 内置 5 分钟缓冲期（在实际过期前返回 true）
   - `refreshOAuthToken()` 使用 refresh token 获取新 access token
   - 刷新失败时回退到重新登录流程
3. **检查 PKCE 参数**：
   - `code_verifier` 每次 OAuth 流程唯一生成（`crypto.generateCodeVerifier()`）
   - `code_challenge_method=S256` 使用 SHA-256 哈希
   - PKCE 参数不匹配会导致 token exchange 失败
4. **检查 redirect URI**：
   - 本地 HTTP 服务器监听 localhost 端口
   - 端口占用时自动尝试其他端口（port 0 让 OS 分配）
   - state 参数不匹配会触发 CSRF 防护拒绝回调

**检查认证源优先级**：
```
1. 3P context       → 第三方集成凭证（最高优先级）
2. bare mode        → 精简模式特殊认证
3. managed OAuth    → 托管环境 OAuth
4. explicit tokens  → ANTHROPIC_AUTH_TOKEN / apiKeyHelper / FILE_DESCRIPTOR
5. OAuth            → 标准 OAuth PKCE 流程
6. API key fallback → ANTHROPIC_API_KEY（最低优先级）
```

### 添加新认证方式

**在 `getAnthropicClient()` 中添加认证分支：**

1. 在 `src/services/api/client.ts` 的 `getAnthropicClient()` 函数中添加新的认证分支
2. 实现 token 获取逻辑（同步或异步）
3. 按优先级顺序插入认证解析链
4. 确保新认证方式也支持自动刷新（如果适用）

### Token 生命周期管理

**自动刷新机制：**

- `isOAuthTokenExpired()` 内置 **5 分钟缓冲期**，确保有足够时间完成刷新
- `jwtUtils.ts` 中的 `createTokenRefreshScheduler` 实现主动刷新——根据 token 过期时间提前调度
- 刷新失败时回退到完整 OAuth 登录流程

**Keychain 缓存（macOS）：**

- `KEYCHAIN_CACHE_TTL_MS = 5 * 60 * 1000`（5 分钟）减少对系统 Keychain 的频繁访问
- **Stale-while-error 策略**：token 刷新失败时，如果缓存有旧值仍使用旧值继续工作
- 凭证通过 **hex 编码** 存储，规避进程监控工具的明文捕获
- stdin 传输限制 **4032 bytes**（`SECURITY_STDIN_LINE_LIMIT`），超过时降级为 argv 传参

### 安全存储平台差异

| 平台 | 存储后端 | 安全级别 | 降级方案 |
|------|---------|---------|---------|
| macOS | Keychain（`security` CLI） | 高（OS 级加密） | plaintext（`~/.claude/.credentials.json`） |
| Windows | plaintext | 中（文件权限 0600） | 无 |
| Linux | plaintext | 中（文件权限 0600） | 无 |

### 常见陷阱

| 陷阱 | 详情 | 解决方案 |
|------|------|----------|
| OAuth token 存在 keychain——卸载不会清除 | macOS Keychain 中的凭证在卸载 Claude Code 后仍然存在 | 手动使用 `security delete-generic-password` 清除 |
| refresh_token 过期需要重新授权 | refresh token 有自己的有效期，过期后无法自动刷新 | 系统自动回退到完整 OAuth 登录流程 |
| Keychain 锁定时降级 | macOS Keychain 锁定状态下无法读写 | `isMacOsKeychainLocked()` 检测后自动降级到 plaintext |
| 端口冲突 | 本地 OAuth callback 服务器端口被占用 | 使用 port 0 让 OS 分配随机可用端口 |
| state 参数 CSRF 防护 | 回调中 state 不匹配会被拒绝 | 确保浏览器回调的 state 参数与发起请求时一致 |
| `getAnthropicClient` 循环引用风险 | 此函数被多处调用 | 源码注释："Currently we create a new GoogleAuth instance for every getAnthropicClient() call"——注意性能影响 |


---

[← 插件系统](../21-插件系统/plugin-system.md) | [目录](../README.md) | [LSP 集成 →](../23-LSP集成/lsp-integration.md)
