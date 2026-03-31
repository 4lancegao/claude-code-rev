# 账户验证机制调研报告

## 概述

Claude Code CLI 的账户验证体系涉及 OAuth 2.0 认证、API Key 验证、凭证安全存储、订阅权限检查等多个层面。

---

## 1. OAuth 2.0 + PKCE 认证流程（主要方式）

### 核心文件
- `src/services/oauth/client.ts` — OAuth 客户端实现
- `src/services/oauth/index.ts` — OAuthService 编排类
- `src/services/oauth/auth-code-listener.ts` — 本地回调监听器
- `src/constants/oauth.ts` — OAuth 配置（生产/预发/本地）

### 流程
1. 生成 PKCE code verifier 和 code challenge
2. 构建授权 URL，包含 state 参数（防 CSRF 攻击）
3. 打开浏览器跳转至 OAuth 授权页面
4. 本地启动临时 HTTP 服务器（`AuthCodeListener`），监听回调
5. 回调中验证 state 参数是否匹配
6. 用 authorization code 调用 `exchangeCodeForTokens()` 换取 access/refresh token
7. Token 存入安全存储

### OAuth Scopes
- `user:inference` — Claude.ai 推理访问
- `user:profile` — 用户资料
- `user:sessions:claude_code` — 会话管理
- `user:mcp_servers` — MCP 服务器访问
- `user:file_upload` — 文件上传
- `org:create_api_key` — Console API Key 创建

### 关键函数
| 函数 | 文件 | 说明 |
|------|------|------|
| `buildAuthUrl()` | `oauth/client.ts:46-105` | 构建 OAuth 授权 URL |
| `exchangeCodeForTokens()` | `oauth/client.ts:107-144` | Authorization code 换 token |
| `refreshOAuthToken()` | `oauth/client.ts:146-274` | 刷新过期 token |
| `isOAuthTokenExpired()` | `oauth/client.ts:344-352` | 检查 token 是否过期（5分钟缓冲） |

---

## 2. API Key 验证

### 核心文件
- `src/services/api/claude.ts:530-580` — `verifyApiKey()` 函数
- `src/services/oauth/client.ts:311-342` — `createAndStoreApiKey()`

### 验证机制
- 用 API Key 向 Haiku 模型发起一次测试请求
- 最多重试 3 次（指数退避）
- 检查响应中是否包含认证错误
- 非交互模式下跳过验证以提升性能

### API Key 创建
- 通过 OAuth access token POST 至 `/api/oauth/claude_cli/create_api_key`
- 返回 `raw_key` 并存入安全存储

---

## 3. 凭证安全存储

### 核心文件
- `src/utils/secureStorage/index.ts` — 存储抽象层
- `src/utils/secureStorage/macOsKeychainStorage.ts` — macOS Keychain 实现
- `src/utils/secureStorage/plainTextStorage.ts` — 明文降级方案
- `src/utils/secureStorage/fallbackStorage.ts` — 降级策略

### 平台实现
| 平台 | 存储方式 | 特点 |
|------|----------|------|
| macOS | 系统 Keychain（`security` 命令） | 5 分钟缓存，瞬态错误时回退到缓存 |
| Linux/Windows | `~/.claude/.credentials.json` 明文 | 降级方案，TODO: 未来支持 libsecret |

### 接口
```typescript
interface SecureStorage {
  get(key: string): Promise<SecureStorageData | null>
  set(key: string, value: SecureStorageData): Promise<void>
  delete(key: string): Promise<void>
}
```

---

## 4. 账户信息获取与存储

### 核心文件
- `src/utils/config.ts:161-174` — `AccountInfo` 类型定义
- `src/services/oauth/getOauthProfile.ts` — 获取 OAuth 用户资料

### AccountInfo 结构
```typescript
type AccountInfo = {
  accountUuid: string
  emailAddress: string
  organizationUuid?: string
  organizationName?: string
  organizationRole?: string
  workspaceRole?: string
  displayName?: string
  hasExtraUsageEnabled?: boolean
  billingType?: 'stripe_subscription' | 'apple_subscription' | 'google_play_subscription' | ...
  accountCreatedAt?: string   // ISO timestamp
  subscriptionCreatedAt?: string
}
```

### 获取方式
- 端点: `{BASE_API_URL}/api/oauth/profile`
- Beta Header: `anthropic-beta: oauth-2025-04-20`
- 两种获取方式:
  - `getOauthProfileFromOauthToken(accessToken)` — 使用 OAuth token
  - `getOauthProfileFromApiKey()` — 使用 API key + account UUID

### 存储
- 持久化到 `~/.claude/config.json` 的 `oauthAccount` 字段

---

## 5. Token 刷新与并发控制

### 核心文件
- `src/utils/auth.ts:1427-1562` — `checkAndRefreshOAuthTokenIfNeeded()`

### 机制
1. 检查 `tokens.expiresAt`，5 分钟缓冲期内触发刷新
2. 获取文件级锁，防止多进程同时刷新
3. 锁内再次检查 token 是否已被其他进程刷新
4. 调用 `refreshOAuthToken()` 执行刷新
5. 刷新后清除 Keychain 缓存
6. 最多 5 次重试（指数退避处理锁竞争）

### Scope 检查
- `isClaudeAISubscriber()` — 检查 `user:inference` scope
- `hasProfileScope()` — 检查 `user:profile` scope

---

## 6. 订阅 & 权限验证

### 核心文件
- `src/utils/auth.ts:1623-1690`

### 关键函数
| 函数 | 返回值 | 说明 |
|------|--------|------|
| `getSubscriptionType()` | `'pro' \| 'max' \| 'team' \| 'enterprise' \| null` | 获取订阅类型 |
| `hasOpusAccess()` | `boolean` | 是否有 Opus 模型访问权限 |
| `isOverageProvisioningAllowed()` | `boolean` | 是否允许超额使用 |

### 超额使用条件
需同时满足:
- 是 Claude.ai 订阅者
- billing type 为 `stripe_subscription`、`stripe_subscription_contracted`、`apple_subscription` 或 `google_play_subscription`

---

## 7. 企业 SSO / MCP OAuth

### 核心文件
- `src/services/mcp/xaaIdpLogin.ts` — 企业 IdP 登录
- `src/services/mcp/auth.ts` — MCP OAuth 流程

### 支持的 IdP
- Azure AD
- Okta
- Keycloak
- 其他标准 OIDC 提供商

### 流程
1. OIDC 自动发现（`.well-known/openid-configuration`）
2. PKCE + authorization code 流程获取 id_token
3. JWT exp claim 校验缓存有效期
4. Token 存入安全存储（按 issuer 区分 key）

### 安全措施
- State 参数防 CSRF
- 强制 HTTPS token 端点
- OIDC metadata schema 验证

---

## 8. 认证方法判断

### 核心文件
- `src/cli/handlers/auth.ts:232-319` — `authStatus()`
- `src/utils/auth.ts` — `getAuthTokenSource()`

### 认证方式优先级
| 方式 | 标识 | 来源 |
|------|------|------|
| Claude.ai OAuth | `'claude.ai'` | OAuth 登录 |
| Console OAuth | `'oauth_token'` | Console OAuth 登录 |
| API Key Helper | `'api_key_helper'` | 辅助脚本提供 |
| 直接 API Key | `'api_key'` | 环境变量或配置 |
| 第三方服务 | `'third_party'` | Vertex、Bedrock 等 |
| 未认证 | `'none'` | — |

### 托管 OAuth 上下文
- `isManagedOAuthContext()` 检查 `CLAUDE_CODE_REMOTE` 或 `CLAUDE_CODE_ENTRYPOINT === 'claude-desktop'`
- 在托管会话中阻止回退到终端 API Key

---

## 9. 信任对话框验证

### 核心文件
- `src/utils/config.ts:697-761` — `checkHasTrustDialogAccepted()`

### 机制
- 从当前工作目录向上遍历目录树
- 检查任一祖先目录是否有 `hasTrustDialogAccepted: true`
- 支持会话级（内存）和项目级（持久化）信任
- 防止通过恶意 `settings.json` 进行 RCE 攻击

---

## 10. 登录/登出命令

### 登录 (`authLogin()`)
- **文件**: `src/cli/handlers/auth.ts:112-230`
- **选项**:
  - `--console` — Console OAuth 登录
  - `--claudeai` — Claude.ai OAuth 登录
  - `--email` — 预填登录邮箱
  - `--sso` — 强制 SSO 方式
- **环境变量捷径**:
  - `CLAUDE_CODE_OAUTH_REFRESH_TOKEN` — 跳过浏览器流程
  - `CLAUDE_CODE_OAUTH_SCOPES` — 配合 refresh token 使用

### 登出 (`authLogout()`)
- **文件**: `src/cli/handlers/auth.ts:321-330`
- 清除: OAuth tokens、API keys、账户信息缓存、betas 缓存、tool schemas 缓存

---

## 11. 遥测事件

### 关键认证事件
| 事件名 | 说明 |
|--------|------|
| `tengu_oauth_flow_start` | OAuth 流程启动 |
| `tengu_oauth_token_exchange_success` | Auth code 换 token 成功 |
| `tengu_oauth_token_refresh_success` | Token 刷新成功 |
| `tengu_oauth_token_refresh_failure` | Token 刷新失败 |
| `tengu_oauth_api_key` | API Key 创建结果 |
| `tengu_oauth_success` | 完整登录流程完成 |
| `tengu_login_from_refresh_token` | 环境变量 token 登录 |

---

## 架构总览

```
用户登录
  ├── Claude.ai OAuth (PKCE) → access/refresh token → Keychain/文件存储
  ├── Console OAuth → 创建 API Key → 安全存储
  ├── 企业 SSO (OIDC) → id_token → 安全存储
  └── 直接 API Key → 测试调用验证

Token 生命周期管理
  ├── 过期检测 (5分钟缓冲)
  ├── 自动刷新 (文件锁 + 指数退避)
  ├── Scope 检查 (inference, profile 等)
  └── 并发控制 (文件锁防竞争)

权限层级
  ├── 订阅类型 (pro/max/team/enterprise)
  ├── 模型访问权限 (Opus/Sonnet/Haiku)
  ├── 功能开关 (overage, MCP servers 等)
  └── 项目信任 (目录树遍历)
```
