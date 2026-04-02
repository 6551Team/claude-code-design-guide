# OAuth and Authentication System

Claude Code's authentication system implements a complete OAuth 2.0 PKCE flow, secure token storage, and a multi-source authentication resolution chain.

---

## OAuth Service (src/services/oauth/)

### Design Philosophy

#### Why PKCE Instead of Simple OAuth?

Claude Code is a desktop CLI application and therefore a "public client" in OAuth terminology — it cannot securely store a client_secret (any secret embedded in a local binary can be extracted via reverse engineering). PKCE (Proof Key for Code Exchange) is specifically designed for this scenario: it replaces the client_secret with a one-time `code_verifier`/`code_challenge` pair, so even if the authorization code is intercepted, it cannot be exchanged for a token. In the `OAuthService` constructor, `this.codeVerifier = crypto.generateCodeVerifier()` generates a unique verifier for each flow, and `code_challenge_method=S256` in `client.ts` uses SHA-256 hashing to ensure the verifier is never transmitted in plaintext.

### OAuthService Class

The core OAuth service class that manages the complete authentication flow.

#### Main Methods

**startOAuthFlow()**
- Initiates the complete OAuth authentication flow
- Flow: localhost listener → PKCE + state → build URLs → wait for code → exchange → profile fetch
- Automatically opens the browser to the authorization page
- Starts a local HTTP server to listen for the callback

**handleManualAuthCodeInput()**
- Supports manual paste of the authorization code when the automatic flow fails (e.g., browser does not open)
- Parses the user-provided authorization code and continues the token exchange flow

**cleanup()**
- Cleans up temporary resources from the OAuth flow
- Closes the localhost HTTP listener
- Clears temporary state

### OAuth Flow — Detailed Steps

1. Generate PKCE code_verifier and code_challenge
2. Generate a random state parameter to prevent CSRF
3. Start a local HTTP server to listen for the callback
4. Build the authorization URL and open the browser
5. Wait for the callback after user authorization (carrying the authorization code)
6. Exchange tokens using code + code_verifier
7. Fetch the user profile information

---

## Token Exchange (client.ts)

### buildAuthUrl

```typescript
buildAuthUrl(codeChallenge: string, state: string) → string
```

Builds the OAuth authorization URL, including:
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

Exchanges the authorization code for an access token and refresh token:
- Sends a POST request to the token endpoint
- Includes code_verifier for PKCE verification
- Has a built-in retry mechanism to handle transient network errors

### refreshOAuthToken

```typescript
refreshOAuthToken(refreshToken: string, scopes?: string[]) → { accessToken, refreshToken, expiresIn }
```

Refreshes the access token using the refresh token:
- Supports specifying a scope to narrow permissions
- Returns a new token pair (access + refresh)

#### Why Automatic Token Refresh?

Users should not have their workflow interrupted due to token expiry — silent refresh is the best UX. `isOAuthTokenExpired()` has a built-in 5-minute buffer (returns `true` before the actual expiry time), ensuring there is enough time to complete the refresh. `createTokenRefreshScheduler` in `jwtUtils.ts` even implements proactive refresh — it schedules the refresh ahead of time based on the token's expiry, rather than waiting passively for a request to fail. Only when the refresh fails does the system fall back to the re-login flow, minimizing disruption to the user's work.

### fetchProfileInfo

```typescript
fetchProfileInfo(accessToken: string) → {
  subscription: string,
  rateLimitTier: string,
  displayName: string,
  billingType: string,
}
```

Fetches the user profile information:
- subscription: subscription type (free, pro, max, etc.)
- rateLimitTier: rate limit tier
- displayName: user display name
- billingType: billing type

### isOAuthTokenExpired

```typescript
isOAuthTokenExpired() → boolean
```

Checks whether the current OAuth token has expired:
- Has a built-in 5-minute buffer, returning true before the actual expiry
- Ensures there is enough time to complete token refresh

### populateOAuthAccountInfoIfNeeded

```typescript
populateOAuthAccountInfoIfNeeded() → void
```

Lazily fetches OAuth account information:
- Only makes a request when needed (e.g., on first access to account information)
- Caches the result after fetching to avoid repeated requests

---

## Auth Code Listener (auth-code-listener.ts)

### AuthCodeListener Class

Starts a local HTTP server to receive OAuth callbacks.

#### start

```typescript
start(port?: number) → Promise<number>
```

- Starts a localhost HTTP server
- If no port is specified, uses an OS-assigned random port (port 0)
- Returns the actual port being listened on

#### waitForAuthorization

```typescript
waitForAuthorization(state: string, onReady: (url: string) => void) → Promise<string>
```

- Waits for the callback request from the OAuth provider
- The `state` parameter is used to validate the legitimacy of the callback
- The `onReady` callback fires when the server is ready, passing in the redirect URL

#### Security Validation and Error Handling

- **State mismatch**: the state in the callback does not match the expected value → reject, preventing CSRF attacks
- **Missing code**: the authorization code is absent from the callback → returns HTTP 400
- **Success redirect**: redirects to a success page after successful authentication
- **Error redirect**: redirects to an error page after failed authentication, displaying error information

---

## Secure Storage (src/utils/secureStorage/)

### Design Philosophy

#### Why Keychain Integration?

Storing tokens in plaintext is a security anti-pattern — any process that can read user files can steal credentials. The OS keychain (macOS Keychain / Windows Credential Manager) provides OS-level encrypted storage, with credentials protected by the user's login password. The macOS implementation (`macOsKeychainStorage.ts`) interacts with Keychain via the `security` CLI tool, preferring to pass credentials via stdin (safer than command-line arguments, not exposed in the process list), and only falling back to argv when the credential exceeds 4032 bytes (`SECURITY_STDIN_LINE_LIMIT`). Credential values are stored using hex encoding to prevent process monitoring tools from capturing plaintext on the command line. The 5-minute `KEYCHAIN_CACHE_TTL_MS` reduces frequent access to the system Keychain, and the stale-while-error strategy allows the system to continue working with cached stale values when token refresh fails — all of these are designed to strike a balance between security and usability.

### Platform Abstraction

Selects a different storage backend based on the running platform:

- **macOS**: keychain storage + plaintext fallback
- **Windows/Linux**: plaintext storage

### plainTextStorage

```
Storage location: ~/.claude/.credentials.json
File permissions: 0600 (owner read/write only)
```

- Stores credentials in JSON format on the filesystem
- Security is protected by strict file permissions
- Serves as the fallback storage for all platforms

### macOsKeychainStorage

Uses the macOS system Keychain for secure storage:

```
Commands: security find-generic-password / security add-generic-password
```

#### KEYCHAIN_CACHE_TTL_MS

```typescript
const KEYCHAIN_CACHE_TTL_MS = 5 * 60 * 1000  // 5 minutes
```

In-memory cache TTL for Keychain read results, reducing frequent access to the system Keychain.

#### Stale-while-error Strategy

When token refresh fails, if there is a stale value in the cache (even if past its TTL), that stale value is still used to continue working rather than failing immediately. This is similar to the HTTP stale-while-revalidate pattern.

#### Hex Encoding

```
Credential values are hex-encoded before being stored in the Keychain
```

Credentials are stored using hexadecimal encoding to prevent process monitoring tools from capturing plaintext credentials in command-line arguments.

#### stdin Limit and Fallback

```
stdin transfer limit: 4032 bytes
Exceeds limit: falls back to argv
```

- Prefers passing credentials to the `security` command via stdin (safer, not exposed in process arguments)
- When credentials exceed 4032 bytes, falls back to passing them via command-line arguments

#### isMacOsKeychainLocked

```typescript
isMacOsKeychainLocked() → boolean
```

Checks whether the macOS Keychain is in a locked state:
- Result is cached to avoid frequent detection
- Automatically falls back to plaintext storage when the Keychain is locked

---

## Auth Resolution Chain

### Authentication Source Priority

The authentication system tries each source in the following priority order, using the first one that succeeds:

```
1. 3P context        → credentials provided by a third-party integration context
2. bare mode         → special authentication in bare mode
3. managed OAuth     → OAuth credentials for managed environments
4. explicit tokens   → explicitly configured tokens
5. OAuth             → tokens obtained via the standard OAuth flow
6. API key fallback  → API key fallback
```

### Authentication Source Details

**3P Context (Third-Party Context)**
- When Claude Code is integrated as a third-party tool, credentials are provided by the host environment
- Highest priority, used directly

**Bare Mode**
- Authentication in slim mode, bypassing the standard flow

**Managed OAuth**
- OAuth credentials pre-configured for enterprise or managed environments

**Explicit Tokens**
- `ANTHROPIC_AUTH_TOKEN` environment variable
- Token obtained via the `apiKeyHelper` external program
- `FILE_DESCRIPTOR` for passing tokens via a file descriptor

**OAuth (claude.ai)**
- Standard OAuth 2.0 PKCE flow
- Token obtained via claude.ai authorization
- Supports automatic refresh

**API Key Fallback**
- `ANTHROPIC_API_KEY` environment variable
- Lowest-priority fallback
- Uses the API key directly rather than an OAuth token

---

## Engineering Practice Guide

### Debugging Authentication Failures

**Troubleshooting checklist:**

1. **Check the token in the keychain**:
   - macOS: `security find-generic-password -s claude-code` to view the keychain entry
   - Windows/Linux: check `~/.claude/.credentials.json` (permissions 0600)
2. **Manually refresh the token**:
   - `isOAuthTokenExpired()` has a built-in 5-minute buffer (returns true before actual expiry)
   - `refreshOAuthToken()` uses the refresh token to obtain a new access token
   - Falls back to the re-login flow when refresh fails
3. **Check PKCE parameters**:
   - `code_verifier` is uniquely generated for each OAuth flow (`crypto.generateCodeVerifier()`)
   - `code_challenge_method=S256` uses SHA-256 hashing
   - Mismatched PKCE parameters will cause token exchange to fail
4. **Check the redirect URI**:
   - The local HTTP server listens on a localhost port
   - Automatically tries other ports when a port is occupied (port 0 lets the OS assign one)
   - A state parameter mismatch will trigger CSRF protection and reject the callback

**Check authentication source priority**:
```
1. 3P context       → third-party integration credentials (highest priority)
2. bare mode        → slim mode special authentication
3. managed OAuth    → managed environment OAuth
4. explicit tokens  → ANTHROPIC_AUTH_TOKEN / apiKeyHelper / FILE_DESCRIPTOR
5. OAuth            → standard OAuth PKCE flow
6. API key fallback → ANTHROPIC_API_KEY (lowest priority)
```

### Adding a New Authentication Method

**Add an authentication branch in `getAnthropicClient()`:**

1. Add a new authentication branch in the `getAnthropicClient()` function in `src/services/api/client.ts`
2. Implement token retrieval logic (synchronous or asynchronous)
3. Insert into the authentication resolution chain in priority order
4. Ensure the new authentication method also supports automatic refresh (if applicable)

### Token Lifecycle Management

**Automatic refresh mechanism:**

- `isOAuthTokenExpired()` has a built-in **5-minute buffer**, ensuring enough time to complete refresh
- `createTokenRefreshScheduler` in `jwtUtils.ts` implements proactive refresh — scheduled ahead of time based on the token's expiry
- Falls back to the full OAuth login flow when refresh fails

**Keychain cache (macOS):**

- `KEYCHAIN_CACHE_TTL_MS = 5 * 60 * 1000` (5 minutes) reduces frequent access to the system Keychain
- **Stale-while-error strategy**: when token refresh fails, if the cache has a stale value, it continues to work with that value
- Credentials are stored with **hex encoding** to prevent process monitoring tools from capturing plaintext
- stdin transfer limit is **4032 bytes** (`SECURITY_STDIN_LINE_LIMIT`); falls back to argv when exceeded

### Secure Storage Platform Differences

| Platform | Storage Backend | Security Level | Fallback |
|----------|----------------|----------------|---------|
| macOS | Keychain (`security` CLI) | High (OS-level encryption) | plaintext (`~/.claude/.credentials.json`) |
| Windows | plaintext | Medium (file permissions 0600) | None |
| Linux | plaintext | Medium (file permissions 0600) | None |

### Common Pitfalls

| Pitfall | Details | Solution |
|---------|---------|---------|
| OAuth token in keychain — not cleared on uninstall | Credentials in the macOS Keychain persist after Claude Code is uninstalled | Manually clear using `security delete-generic-password` |
| Expired refresh_token requires re-authorization | The refresh token has its own validity period; it cannot be auto-refreshed after expiry | The system automatically falls back to the full OAuth login flow |
| Fallback when Keychain is locked | Cannot read/write the macOS Keychain while it is locked | `isMacOsKeychainLocked()` detects this and automatically falls back to plaintext |
| Port conflict | The local OAuth callback server port is occupied | Use port 0 to let the OS assign a random available port |
| state parameter CSRF protection | A mismatched state in the callback will be rejected | Ensure the state parameter in the browser callback matches the one sent in the original request |
| `getAnthropicClient` circular reference risk | This function is called from many places | Source comment: "Currently we create a new GoogleAuth instance for every getAnthropicClient() call" — be aware of the performance impact |


---

[← Plugin System](../21-插件系统/plugin-system-en.md) | [Index](../README_EN.md) | [LSP Integration →](../23-LSP集成/lsp-integration-en.md)
