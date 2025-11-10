# NaviHive 安全修复方案

## 修复优先级和执行计划

本文档详细说明了所有安全问题的修复方案，按优先级排序，每个修复都包含具体实现步骤和代码示例。

---

## 阶段 1: 关键安全问题修复（立即执行）

### ✅ 任务 1.1: 修复 JWT 签名实现 [CR-001]
**优先级**: 🔴 严重
**影响文件**: `src/API/http.ts`
**预计时间**: 2小时

#### 问题描述
当前 JWT 实现使用简单的 base64 编码而非加密签名，任何人都可以伪造有效的 token。

#### 修复方案

**步骤 1**: 替换 JWT 生成逻辑
```typescript
// src/API/http.ts - 替换 generateToken 方法

private async generateToken(payload: Record<string, unknown>): Promise<string> {
  const header = { alg: 'HS256', typ: 'JWT' };

  // 编码 header 和 payload
  const encodedHeader = this.base64UrlEncode(JSON.stringify(header));
  const encodedPayload = this.base64UrlEncode(JSON.stringify(payload));

  // 使用 Web Crypto API 进行 HMAC-SHA256 签名
  const encoder = new TextEncoder();
  const data = encoder.encode(`${encodedHeader}.${encodedPayload}`);
  const keyData = encoder.encode(this.secret);

  // 导入密钥
  const key = await crypto.subtle.importKey(
    'raw',
    keyData,
    { name: 'HMAC', hash: 'SHA-256' },
    false,
    ['sign']
  );

  // 生成签名
  const signatureBuffer = await crypto.subtle.sign('HMAC', key, data);
  const signature = this.base64UrlEncode(signatureBuffer);

  return `${encodedHeader}.${encodedPayload}.${signature}`;
}

// 辅助方法：支持 ArrayBuffer 编码
private base64UrlEncode(data: string | ArrayBuffer): string {
  let base64: string;

  if (typeof data === 'string') {
    base64 = btoa(data);
  } else {
    // ArrayBuffer 转 base64
    const bytes = new Uint8Array(data);
    const binary = Array.from(bytes)
      .map(byte => String.fromCharCode(byte))
      .join('');
    base64 = btoa(binary);
  }

  return base64
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=+$/, '');
}
```

**步骤 2**: 替换 JWT 验证逻辑
```typescript
// src/API/http.ts - 替换 verifyToken 方法

async verifyToken(token: string): Promise<{ valid: boolean; payload?: Record<string, unknown> }> {
  try {
    const parts = token.split('.');
    if (parts.length !== 3) {
      return { valid: false };
    }

    const [encodedHeader, encodedPayload, signature] = parts;

    // 重新生成签名进行验证
    const encoder = new TextEncoder();
    const data = encoder.encode(`${encodedHeader}.${encodedPayload}`);
    const keyData = encoder.encode(this.secret);

    // 导入密钥
    const key = await crypto.subtle.importKey(
      'raw',
      keyData,
      { name: 'HMAC', hash: 'SHA-256' },
      false,
      ['verify']
    );

    // 解码签名
    const signatureBytes = this.base64UrlDecode(signature);

    // 验证签名
    const isValid = await crypto.subtle.verify('HMAC', key, signatureBytes, data);

    if (!isValid) {
      return { valid: false };
    }

    // 解码并验证 payload
    const payloadStr = atob(encodedPayload.replace(/-/g, '+').replace(/_/g, '/'));
    const payload = JSON.parse(payloadStr) as Record<string, unknown>;

    // 检查过期时间
    const now = Math.floor(Date.now() / 1000);
    if (payload.exp && typeof payload.exp === 'number' && payload.exp < now) {
      return { valid: false };
    }

    return { valid: true, payload };
  } catch (error) {
    console.error('Token 验证失败:', error);
    return { valid: false };
  }
}

// 辅助方法：base64url 解码为 ArrayBuffer
private base64UrlDecode(base64url: string): ArrayBuffer {
  const base64 = base64url.replace(/-/g, '+').replace(/_/g, '/');
  const padding = '='.repeat((4 - (base64.length % 4)) % 4);
  const binary = atob(base64 + padding);
  const bytes = new Uint8Array(binary.length);
  for (let i = 0; i < binary.length; i++) {
    bytes[i] = binary.charCodeAt(i);
  }
  return bytes.buffer;
}
```

**步骤 3**: 更新 login 方法使用 async
```typescript
// src/API/http.ts - 更新 login 方法签名

async login(loginRequest: LoginRequest): Promise<LoginResponse> {
  if (!this.authEnabled) {
    return { success: false, message: '身份验证未启用' };
  }

  if (loginRequest.username === this.username && loginRequest.password === this.password) {
    const expiresIn = loginRequest.rememberMe ? 30 * 24 * 60 * 60 : 7 * 24 * 60 * 60;
    const payload = {
      username: loginRequest.username,
      exp: Math.floor(Date.now() / 1000) + expiresIn,
    };

    // 使用 async 版本的 generateToken
    const token = await this.generateToken(payload);

    return {
      success: true,
      token,
      message: '登录成功',
    };
  }

  return { success: false, message: '用户名或密码错误' };
}
```

**验证步骤**:
1. 使用在线 JWT 验证工具（jwt.io）验证生成的 token
2. 尝试修改 token 内容，验证是否被拒绝
3. 测试 token 过期功能

---

### ✅ 任务 1.2: 修复 XSS 漏洞 - 自定义 CSS 注入 [CR-003]
**优先级**: 🔴 严重
**影响文件**: `src/App.tsx`
**预计时间**: 1.5小时

#### 问题描述
当前的正则表达式 CSS 清理可以被绕过，允许注入恶意 JavaScript。

#### 修复方案

**步骤 1**: 实施严格的 CSP（内容安全策略）
```typescript
// worker/index.ts - 在所有响应中添加 CSP 头

// 在 fetch 方法开始处添加
const DEFAULT_HEADERS = {
  'Content-Security-Policy': [
    "default-src 'self'",
    "style-src 'self' 'unsafe-inline'", // 允许内联样式
    "script-src 'self'", // 只允许同源脚本
    "img-src 'self' https: data:", // 允许 HTTPS 图片和 data URLs
    "font-src 'self' https://fonts.gstatic.com",
    "connect-src 'self'",
    "frame-ancestors 'none'",
    "base-uri 'self'",
    "form-action 'self'",
  ].join('; '),
  'X-Content-Type-Options': 'nosniff',
  'X-Frame-Options': 'DENY',
  'X-XSS-Protection': '1; mode=block',
  'Referrer-Policy': 'strict-origin-when-cross-origin',
};

// 辅助函数：添加安全头
function addSecurityHeaders(response: Response): Response {
  const newResponse = new Response(response.body, response);
  Object.entries(DEFAULT_HEADERS).forEach(([key, value]) => {
    newResponse.headers.set(key, value);
  });
  return newResponse;
}

// 在返回响应前应用
return addSecurityHeaders(Response.json(data));
```

**步骤 2**: 增强 CSS 清理逻辑
```typescript
// src/App.tsx - 创建新的 CSS 清理函数

const sanitizeCSS = (css: string): string => {
  if (!css || typeof css !== 'string') return '';

  // 1. 移除所有注释
  let sanitized = css.replace(/\/\*[\s\S]*?\*\//g, '');

  // 2. 白名单允许的 CSS 属性
  const ALLOWED_PROPERTIES = [
    'color', 'background', 'background-color', 'background-image',
    'background-size', 'background-position', 'background-repeat',
    'font-size', 'font-family', 'font-weight', 'font-style',
    'margin', 'padding', 'border', 'border-radius',
    'width', 'height', 'max-width', 'max-height',
    'display', 'flex', 'grid', 'align-items', 'justify-content',
    'text-align', 'line-height', 'letter-spacing',
    'opacity', 'transform', 'transition', 'animation',
  ];

  // 3. 移除危险的 CSS 值
  const DANGEROUS_PATTERNS = [
    /javascript:/gi,
    /data:text\/html/gi,
    /vbscript:/gi,
    /@import/gi,
    /expression\s*\(/gi,
    /-moz-binding/gi,
    /behavior\s*:/gi,
    /<\s*script/gi,
    /<\s*iframe/gi,
  ];

  DANGEROUS_PATTERNS.forEach(pattern => {
    sanitized = sanitized.replace(pattern, '');
  });

  // 4. 清理 url() 中的危险内容
  sanitized = sanitized.replace(
    /url\s*\(\s*(['"]?)(.*?)\1\s*\)/gi,
    (match, quote, url) => {
      // 只允许 https:, data:image/, 相对路径
      if (
        url.startsWith('https://') ||
        url.startsWith('data:image/') ||
        url.startsWith('/')
      ) {
        return `url(${quote}${url}${quote})`;
      }
      return ''; // 移除不安全的 URL
    }
  );

  // 5. 限制 CSS 长度
  const MAX_CSS_LENGTH = 50000; // 50KB
  if (sanitized.length > MAX_CSS_LENGTH) {
    console.warn('自定义 CSS 超过长度限制，已截断');
    sanitized = sanitized.substring(0, MAX_CSS_LENGTH);
  }

  return sanitized;
};
```

**步骤 3**: 使用清理后的 CSS
```typescript
// src/App.tsx - 在 useEffect 中应用清理

useEffect(() => {
  const customCss = configs['site.customCss'];
  if (customCss) {
    // 清理 CSS
    const sanitized = sanitizeCSS(customCss);

    // 应用到页面
    const styleElement = document.getElementById('custom-css-styles');
    if (styleElement) {
      styleElement.textContent = sanitized;
    } else {
      const style = document.createElement('style');
      style.id = 'custom-css-styles';
      style.textContent = sanitized;
      document.head.appendChild(style);
    }
  }

  // 清理函数
  return () => {
    const styleElement = document.getElementById('custom-css-styles');
    if (styleElement) {
      styleElement.remove();
    }
  };
}, [configs]);
```

**验证步骤**:
1. 尝试注入 `javascript:alert(1)` - 应该被移除
2. 尝试注入 `expression(alert(1))` - 应该被移除
3. 尝试注入 `<script>alert(1)</script>` - 应该被移除
4. 验证合法的 CSS 仍然有效

---

### ✅ 任务 1.3: 修复 SSRF 漏洞 - 背景图片 URL [CR-004]
**优先级**: 🔴 严重
**影响文件**: `src/App.tsx`, `src/components/SiteSettingsModal.tsx`
**预计时间**: 1小时

#### 问题描述
背景图片 URL 未验证，可能被用于访问内部网络或本地文件系统。

#### 修复方案

**步骤 1**: 创建 URL 验证工具函数
```typescript
// src/utils/url.ts - 新建文件

/**
 * 验证 URL 是否安全，防止 SSRF 攻击
 */
export function isSecureUrl(url: string): boolean {
  if (!url || typeof url !== 'string') return false;

  try {
    const parsed = new URL(url);

    // 只允许 https 和 data 协议
    if (!['https:', 'data:'].includes(parsed.protocol)) {
      console.warn(`不安全的协议: ${parsed.protocol}`);
      return false;
    }

    // 对于 data URLs，只允许图片
    if (parsed.protocol === 'data:') {
      if (!parsed.href.startsWith('data:image/')) {
        console.warn('Data URL 必须是图片类型');
        return false;
      }
      return true;
    }

    // 对于 https URLs，检查主机名
    const hostname = parsed.hostname.toLowerCase();

    // 禁止访问本地地址
    const BLOCKED_HOSTNAMES = [
      'localhost',
      '127.0.0.1',
      '0.0.0.0',
      '::1',
    ];

    if (BLOCKED_HOSTNAMES.includes(hostname)) {
      console.warn(`禁止访问本地地址: ${hostname}`);
      return false;
    }

    // 禁止访问私有 IP 范围
    const PRIVATE_IP_PATTERNS = [
      /^10\./,                    // 10.0.0.0/8
      /^172\.(1[6-9]|2[0-9]|3[01])\./, // 172.16.0.0/12
      /^192\.168\./,              // 192.168.0.0/16
      /^169\.254\./,              // 169.254.0.0/16 (Link-local)
      /^fe80:/,                   // IPv6 link-local
      /^fc00:/,                   // IPv6 unique local
    ];

    if (PRIVATE_IP_PATTERNS.some(pattern => pattern.test(hostname))) {
      console.warn(`禁止访问私有 IP 地址: ${hostname}`);
      return false;
    }

    return true;
  } catch (error) {
    console.error('URL 验证失败:', error);
    return false;
  }
}

/**
 * 验证图标 URL（支持更多来源）
 */
export function isSecureIconUrl(url: string): boolean {
  if (!url) return true; // 空图标是允许的

  // 图标可以是相对路径
  if (url.startsWith('/')) {
    return true;
  }

  return isSecureUrl(url);
}

/**
 * 从 URL 中提取域名
 */
export function extractDomain(url: string): string | null {
  if (!url) return null;

  try {
    let fullUrl = url;
    if (!/^https?:\/\//i.test(url)) {
      fullUrl = 'http://' + url;
    }
    const parsedUrl = new URL(fullUrl);
    return parsedUrl.hostname;
  } catch {
    const match = url.match(/^(?:https?:\/\/)?(?:[^@\n]+@)?(?:www\.)?([^:/\n?]+)/im);
    return match && match[1] ? match[1] : url;
  }
}
```

**步骤 2**: 在 App.tsx 中应用验证
```typescript
// src/App.tsx - 导入验证函数

import { isSecureUrl, extractDomain } from './utils/url';

// 修改背景图片样式应用逻辑
const backgroundImageUrl = configs['site.backgroundImage'];
const backgroundOpacity = parseFloat(configs['site.backgroundOpacity'] || '0.15');

const backgroundStyle = backgroundImageUrl && isSecureUrl(backgroundImageUrl)
  ? {
      backgroundImage: `url(${backgroundImageUrl})`,
      backgroundSize: 'cover',
      backgroundPosition: 'center',
      backgroundRepeat: 'no-repeat',
    }
  : {};
```

**步骤 3**: 在设置保存时验证
```typescript
// src/components/SiteSettingsModal.tsx - 添加验证

import { isSecureUrl } from '../utils/url';

const handleSave = async () => {
  try {
    // 验证背景图片 URL
    if (tempBackgroundImage && !isSecureUrl(tempBackgroundImage)) {
      setSnackbarMessage('背景图片 URL 不安全，只允许 HTTPS 协议和公网地址');
      setSnackbarOpen(true);
      return;
    }

    // 保存配置...
    await api.setConfig('site.backgroundImage', tempBackgroundImage || '');
    // ...
  } catch (error) {
    handleError('保存设置失败: ' + (error as Error).message);
  }
};
```

**步骤 4**: 在后端添加验证（可选但推荐）
```typescript
// worker/index.ts - 在配置更新时验证

else if (path.startsWith("configs/") && method === "PUT") {
  const key = path.substring("configs/".length);
  const data = (await request.json()) as ConfigInput;

  // 特殊验证：背景图片 URL
  if (key === 'site.backgroundImage' && data.value) {
    try {
      const url = new URL(data.value);
      if (url.protocol !== 'https:' && url.protocol !== 'data:') {
        return Response.json(
          {
            success: false,
            message: '背景图片只允许 HTTPS 协议',
          },
          { status: 400 }
        );
      }
    } catch {
      return Response.json(
        {
          success: false,
          message: '无效的图片 URL',
        },
        { status: 400 }
      );
    }
  }

  const result = await api.setConfig(key, data.value);
  return Response.json({ success: result });
}
```

**验证步骤**:
1. 尝试使用 `http://` URL - 应该被拒绝
2. 尝试使用 `file:///` URL - 应该被拒绝
3. 尝试使用私有 IP `http://192.168.1.1` - 应该被拒绝
4. 使用有效的 `https://` URL - 应该成功

---

### ✅ 任务 1.4: 修复 SQL 注入风险 [CR-002]
**优先级**: 🔴 严重
**影响文件**: `src/API/http.ts`
**预计时间**: 1小时

#### 问题描述
虽然使用了参数化查询，但字段名是动态拼接的，如果验证被绕过可能导致 SQL 注入。

#### 修复方案

**步骤 1**: 为所有更新操作添加字段白名单
```typescript
// src/API/http.ts - 修改 updateGroup 方法

async updateGroup(id: number, group: Partial<Group>): Promise<Group | null> {
  // 字段白名单
  const ALLOWED_FIELDS = ['name', 'order_num'] as const;
  type AllowedField = typeof ALLOWED_FIELDS[number];

  const updates: string[] = [];
  const params: unknown[] = [];

  // 验证并构建更新语句
  Object.entries(group).forEach(([key, value]) => {
    // 只允许白名单中的字段
    if (ALLOWED_FIELDS.includes(key as AllowedField)) {
      updates.push(`${key} = ?`);
      params.push(value);
    } else if (key !== 'id' && key !== 'created_at' && key !== 'updated_at') {
      console.warn(`尝试更新不允许的字段: ${key}`);
    }
  });

  if (updates.length === 0) {
    throw new Error('没有可更新的字段');
  }

  // 添加 updated_at
  updates.push('updated_at = CURRENT_TIMESTAMP');
  params.push(id);

  const query = `UPDATE groups SET ${updates.join(', ')} WHERE id = ? RETURNING *`;

  const result = await this.db.prepare(query).bind(...params).first<Group>();
  return result || null;
}
```

**步骤 2**: 修改 updateSite 方法
```typescript
// src/API/http.ts - 修改 updateSite 方法

async updateSite(id: number, site: Partial<Site>): Promise<Site | null> {
  // 字段白名单
  const ALLOWED_FIELDS = [
    'group_id',
    'name',
    'url',
    'icon',
    'description',
    'notes',
    'order_num',
  ] as const;
  type AllowedField = typeof ALLOWED_FIELDS[number];

  const updates: string[] = [];
  const params: unknown[] = [];

  Object.entries(site).forEach(([key, value]) => {
    if (ALLOWED_FIELDS.includes(key as AllowedField)) {
      updates.push(`${key} = ?`);
      params.push(value);
    } else if (key !== 'id' && key !== 'created_at' && key !== 'updated_at') {
      console.warn(`尝试更新不允许的字段: ${key}`);
    }
  });

  if (updates.length === 0) {
    throw new Error('没有可更新的字段');
  }

  updates.push('updated_at = CURRENT_TIMESTAMP');
  params.push(id);

  const query = `UPDATE sites SET ${updates.join(', ')} WHERE id = ? RETURNING *`;

  const result = await this.db.prepare(query).bind(...params).first<Site>();
  return result || null;
}
```

**步骤 3**: 添加 SQL 注入测试保护
```typescript
// src/API/http.ts - 添加辅助函数

/**
 * 验证标识符（表名、字段名）是否安全
 */
private isValidIdentifier(identifier: string): boolean {
  // 只允许字母、数字和下划线
  return /^[a-zA-Z_][a-zA-Z0-9_]*$/.test(identifier);
}

/**
 * 安全地构建 ORDER BY 子句
 */
private buildOrderBy(field: string, direction: 'ASC' | 'DESC' = 'ASC'): string {
  if (!this.isValidIdentifier(field)) {
    throw new Error('无效的排序字段');
  }
  if (!['ASC', 'DESC'].includes(direction)) {
    throw new Error('无效的排序方向');
  }
  return `ORDER BY ${field} ${direction}`;
}
```

**验证步骤**:
1. 尝试更新不存在的字段 - 应该被忽略
2. 尝试注入 SQL 代码到字段名 - 应该被拒绝
3. 正常的更新操作仍然有效

---

## 阶段 2: 高优先级修复（1周内完成）

### ✅ 任务 2.1: 将 Token 移至 HttpOnly Cookies [HS-001]
**优先级**: 🟠 高
**影响文件**: `src/API/client.ts`, `worker/index.ts`
**预计时间**: 2小时

#### 修复方案

**步骤 1**: 修改后端设置 Cookie
```typescript
// worker/index.ts - 修改登录响应

if (path === "login" && method === "POST") {
  const loginData = (await request.json()) as LoginInput;
  const validation = validateLogin(loginData);

  if (!validation.valid) {
    return Response.json(
      { success: false, message: `验证失败: ${validation.errors?.join(", ")}` },
      { status: 400 }
    );
  }

  const result = await api.login(loginData as LoginRequest);

  if (result.success && result.token) {
    // 设置 HttpOnly Cookie
    const maxAge = loginData.rememberMe ? 30 * 24 * 60 * 60 : 7 * 24 * 60 * 60;

    return Response.json(
      { success: true, message: result.message },
      {
        headers: {
          'Set-Cookie': [
            `auth_token=${result.token}`,
            'HttpOnly',
            'Secure', // 仅在 HTTPS 下发送
            'SameSite=Strict', // 防止 CSRF
            `Max-Age=${maxAge}`,
            'Path=/',
          ].join('; '),
        },
      }
    );
  }

  return Response.json(result);
}
```

**步骤 2**: 修改前端 Client
```typescript
// src/API/client.ts - 移除 localStorage 操作

export class NavigationClient {
  private baseUrl: string;

  constructor(baseUrl: string = '/api') {
    this.baseUrl = baseUrl;
    // 不再从 localStorage 读取 token
  }

  // 移除 setToken 和 clearToken 方法
  // Cookie 由浏览器自动管理

  async login(username: string, password: string, rememberMe?: boolean): Promise<LoginResponse> {
    const response = await fetch(`${this.baseUrl}/login`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      credentials: 'include', // 重要：包含 Cookie
      body: JSON.stringify({ username, password, rememberMe }),
    });

    const result = await response.json();

    // 不再需要手动存储 token
    // Cookie 会自动存储和发送

    return result;
  }

  async logout(): Promise<void> {
    // 调用后端清除 Cookie
    await fetch(`${this.baseUrl}/logout`, {
      method: 'POST',
      credentials: 'include',
    });
  }

  private async request(endpoint: string, options: RequestInit = {}) {
    const response = await fetch(`${this.baseUrl}/${endpoint}`, {
      ...options,
      credentials: 'include', // 自动包含 Cookie
      headers: {
        'Content-Type': 'application/json',
        ...options.headers,
      },
    });

    if (!response.ok) {
      const error = await response.text();
      throw new Error(error || '请求失败');
    }

    return response.json();
  }
}
```

**步骤 3**: 添加登出接口
```typescript
// worker/index.ts - 添加登出路由

// 登出路由
else if (path === "logout" && method === "POST") {
  return new Response(JSON.stringify({ success: true }), {
    headers: {
      'Content-Type': 'application/json',
      // 清除 Cookie
      'Set-Cookie': [
        'auth_token=',
        'HttpOnly',
        'Secure',
        'SameSite=Strict',
        'Max-Age=0', // 立即过期
        'Path=/',
      ].join('; '),
    },
  });
}
```

**步骤 4**: 修改认证中间件读取 Cookie
```typescript
// worker/index.ts - 修改认证中间件

if (api.isAuthEnabled()) {
  // 从 Cookie 中读取 token
  const cookieHeader = request.headers.get("Cookie");
  let token: string | null = null;

  if (cookieHeader) {
    const cookies = cookieHeader.split(';').reduce((acc, cookie) => {
      const [key, value] = cookie.trim().split('=');
      acc[key] = value;
      return acc;
    }, {} as Record<string, string>);

    token = cookies['auth_token'];
  }

  // 如果没有 token，返回401
  if (!token) {
    return new Response("请先登录", {
      status: 401,
      headers: {
        "WWW-Authenticate": "Bearer",
      },
    });
  }

  // 验证 token
  const verifyResult = await api.verifyToken(token);
  if (!verifyResult.valid) {
    return new Response("认证已过期或无效，请重新登录", { status: 401 });
  }
}
```

---

### ✅ 任务 2.2: 实施登录速率限制 [HS-002]
**优先级**: 🟠 高
**影响文件**: `worker/index.ts`
**预计时间**: 2小时

#### 修复方案

**步骤 1**: 添加 KV 绑定到 wrangler.jsonc
```jsonc
// wrangler.jsonc

{
  // ... 其他配置
  "kv_namespaces": [
    {
      "binding": "RATE_LIMIT",
      "id": "你的KV命名空间ID",
      "preview_id": "你的预览KV命名空间ID"
    }
  ]
}
```

**步骤 2**: 创建速率限制中间件
```typescript
// worker/index.ts - 添加环境变量接口

interface Env {
  DB: D1Database;
  RATE_LIMIT?: KVNamespace; // 添加 KV 绑定
  AUTH_ENABLED?: string;
  AUTH_USERNAME?: string;
  AUTH_PASSWORD?: string;
  AUTH_SECRET?: string;
}

// 速率限制配置
const RATE_LIMIT_CONFIG = {
  MAX_ATTEMPTS: 5, // 最大尝试次数
  WINDOW_MS: 60 * 1000, // 时间窗口（1分钟）
  BLOCK_DURATION_MS: 15 * 60 * 1000, // 封禁时长（15分钟）
};

// 速率限制函数
async function checkRateLimit(
  env: Env,
  identifier: string, // 通常是 IP 地址
  action: string // 例如 "login"
): Promise<{ allowed: boolean; retryAfter?: number }> {
  if (!env.RATE_LIMIT) {
    console.warn('KV 未配置，跳过速率限制');
    return { allowed: true };
  }

  const key = `rate_limit:${action}:${identifier}`;
  const now = Date.now();

  // 获取当前记录
  const record = await env.RATE_LIMIT.get(key, 'json') as {
    attempts: number;
    windowStart: number;
    blockedUntil?: number;
  } | null;

  // 检查是否在封禁期
  if (record?.blockedUntil && record.blockedUntil > now) {
    return {
      allowed: false,
      retryAfter: Math.ceil((record.blockedUntil - now) / 1000),
    };
  }

  // 检查是否在新的时间窗口
  const isNewWindow = !record || (now - record.windowStart) > RATE_LIMIT_CONFIG.WINDOW_MS;

  if (isNewWindow) {
    // 新窗口，重置计数
    await env.RATE_LIMIT.put(
      key,
      JSON.stringify({
        attempts: 1,
        windowStart: now,
      }),
      { expirationTtl: Math.ceil(RATE_LIMIT_CONFIG.BLOCK_DURATION_MS / 1000) }
    );
    return { allowed: true };
  }

  // 在当前窗口内，增加计数
  const newAttempts = record.attempts + 1;

  if (newAttempts > RATE_LIMIT_CONFIG.MAX_ATTEMPTS) {
    // 超过限制，封禁
    const blockedUntil = now + RATE_LIMIT_CONFIG.BLOCK_DURATION_MS;
    await env.RATE_LIMIT.put(
      key,
      JSON.stringify({
        ...record,
        attempts: newAttempts,
        blockedUntil,
      }),
      { expirationTtl: Math.ceil(RATE_LIMIT_CONFIG.BLOCK_DURATION_MS / 1000) }
    );

    return {
      allowed: false,
      retryAfter: Math.ceil(RATE_LIMIT_CONFIG.BLOCK_DURATION_MS / 1000),
    };
  }

  // 更新计数
  await env.RATE_LIMIT.put(
    key,
    JSON.stringify({
      ...record,
      attempts: newAttempts,
    }),
    { expirationTtl: Math.ceil(RATE_LIMIT_CONFIG.BLOCK_DURATION_MS / 1000) }
  );

  return { allowed: true };
}

// 获取客户端 IP
function getClientIP(request: Request): string {
  return (
    request.headers.get('CF-Connecting-IP') ||
    request.headers.get('X-Forwarded-For')?.split(',')[0] ||
    'unknown'
  );
}
```

**步骤 3**: 在登录路由中应用速率限制
```typescript
// worker/index.ts - 修改登录路由

if (path === "login" && method === "POST") {
  // 获取客户端 IP
  const clientIP = getClientIP(request);

  // 检查速率限制
  const rateLimitCheck = await checkRateLimit(env, clientIP, 'login');

  if (!rateLimitCheck.allowed) {
    return Response.json(
      {
        success: false,
        message: `登录尝试过多，请在 ${rateLimitCheck.retryAfter} 秒后重试`,
      },
      {
        status: 429,
        headers: {
          'Retry-After': String(rateLimitCheck.retryAfter || 900),
        },
      }
    );
  }

  const loginData = (await request.json()) as LoginInput;
  const validation = validateLogin(loginData);

  if (!validation.valid) {
    return Response.json(
      { success: false, message: `验证失败: ${validation.errors?.join(", ")}` },
      { status: 400 }
    );
  }

  const result = await api.login(loginData as LoginRequest);

  // 如果登录失败，不增加额外计数（已经在 checkRateLimit 中计数）
  // 如果登录成功，可以选择清除速率限制记录
  if (result.success && env.RATE_LIMIT) {
    await env.RATE_LIMIT.delete(`rate_limit:login:${clientIP}`);
  }

  return Response.json(result);
}
```

**步骤 4**: 创建 KV 命名空间（部署前执行）
```bash
# 创建生产环境 KV
wrangler kv:namespace create "RATE_LIMIT"

# 创建预览环境 KV
wrangler kv:namespace create "RATE_LIMIT" --preview
```

---

### ✅ 任务 2.3: 使用 bcrypt 哈希密码 [HS-003]
**优先级**: 🟠 高
**影响文件**: `src/API/http.ts`, `package.json`
**预计时间**: 1.5小时

#### 修复方案

**步骤 1**: 安装 bcryptjs
```bash
pnpm add bcryptjs
pnpm add -D @types/bcryptjs
```

**步骤 2**: 修改 NavigationAPI 类
```typescript
// src/API/http.ts - 导入 bcrypt

import bcrypt from 'bcryptjs';

export class NavigationAPI {
  private db: D1Database;
  private authEnabled: boolean;
  private username: string;
  private passwordHash: string; // 改为存储哈希值
  private secret: string;

  constructor(env: Env) {
    this.db = env.DB;
    this.authEnabled = env.AUTH_ENABLED === 'true';
    this.username = env.AUTH_USERNAME || '';

    // 注意：在 Workers 中，env.AUTH_PASSWORD 应该已经是哈希值
    // 初次部署时需要手动生成哈希值
    this.passwordHash = env.AUTH_PASSWORD || '';

    this.secret = env.AUTH_SECRET || 'DefaultSecretKey';

    // 验证配置
    if (this.authEnabled && (!this.username || !this.passwordHash)) {
      console.warn('认证已启用但缺少用户名或密码哈希');
    }
  }

  async login(loginRequest: LoginRequest): Promise<LoginResponse> {
    if (!this.authEnabled) {
      return { success: false, message: '身份验证未启用' };
    }

    // 验证用户名
    if (loginRequest.username !== this.username) {
      // 使用恒定时间比较防止时序攻击
      await bcrypt.compare(loginRequest.password, this.passwordHash);
      return { success: false, message: '用户名或密码错误' };
    }

    // 验证密码（使用 bcrypt）
    const isPasswordValid = await bcrypt.compare(
      loginRequest.password,
      this.passwordHash
    );

    if (!isPasswordValid) {
      return { success: false, message: '用户名或密码错误' };
    }

    // 生成 token
    const expiresIn = loginRequest.rememberMe ? 30 * 24 * 60 * 60 : 7 * 24 * 60 * 60;
    const payload = {
      username: loginRequest.username,
      exp: Math.floor(Date.now() / 1000) + expiresIn,
    };

    const token = await this.generateToken(payload);

    return {
      success: true,
      token,
      message: '登录成功',
    };
  }
}
```

**步骤 3**: 创建密码哈希生成工具
```typescript
// scripts/hash-password.ts - 新建文件

import bcrypt from 'bcryptjs';

async function hashPassword(password: string): Promise<string> {
  const saltRounds = 10;
  const hash = await bcrypt.hash(password, saltRounds);
  return hash;
}

// 从命令行参数读取密码
const password = process.argv[2];

if (!password) {
  console.error('用法: pnpm exec ts-node scripts/hash-password.ts <密码>');
  process.exit(1);
}

hashPassword(password).then(hash => {
  console.log('密码哈希值:');
  console.log(hash);
  console.log('\n请将此哈希值设置为 wrangler.jsonc 中的 AUTH_PASSWORD');
});
```

**步骤 4**: 在 package.json 添加脚本
```json
// package.json

{
  "scripts": {
    // ... 其他脚本
    "hash-password": "ts-node scripts/hash-password.ts"
  }
}
```

**步骤 5**: 更新配置说明
```bash
# 生成密码哈希
pnpm hash-password "your-secure-password"

# 输出示例：
# $2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy

# 将生成的哈希值更新到 wrangler.jsonc
```

**步骤 6**: 更新 wrangler.jsonc
```jsonc
// wrangler.jsonc

{
  "vars": {
    "AUTH_ENABLED": "true",
    "AUTH_USERNAME": "admin",
    "AUTH_PASSWORD": "$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy", // 使用哈希值
    "AUTH_SECRET": "your-random-secret-key"
  }
}
```

---

### ✅ 任务 2.4: 添加 CORS 配置 [HS-004]
**优先级**: 🟠 高
**影响文件**: `worker/index.ts`
**预计时间**: 1小时

#### 修复方案

**步骤 1**: 创建 CORS 工具函数
```typescript
// worker/index.ts - 添加 CORS 配置

const ALLOWED_ORIGINS = [
  'https://your-domain.com',
  // 开发环境
  'http://localhost:5173',
  'http://localhost:8788',
];

/**
 * 获取 CORS 头
 */
function getCorsHeaders(request: Request): Record<string, string> {
  const origin = request.headers.get('Origin');
  const isAllowedOrigin = origin && (
    ALLOWED_ORIGINS.includes(origin) ||
    origin.endsWith('.workers.dev') // 允许 Cloudflare Workers 子域名
  );

  return {
    'Access-Control-Allow-Origin': isAllowedOrigin ? origin : ALLOWED_ORIGINS[0],
    'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
    'Access-Control-Allow-Headers': 'Content-Type, Authorization',
    'Access-Control-Allow-Credentials': 'true', // 允许携带 Cookie
    'Access-Control-Max-Age': '86400', // 预检请求缓存24小时
  };
}

/**
 * 处理预检请求
 */
function handleOptions(request: Request): Response {
  return new Response(null, {
    status: 204,
    headers: getCorsHeaders(request),
  });
}
```

**步骤 2**: 在主 fetch 方法中应用 CORS
```typescript
// worker/index.ts - 修改 fetch 方法

export default {
  async fetch(request: Request, env: Env) {
    const url = new URL(request.url);

    // 处理 CORS 预检请求
    if (request.method === 'OPTIONS') {
      return handleOptions(request);
    }

    const corsHeaders = getCorsHeaders(request);

    // API路由处理
    if (url.pathname.startsWith("/api/")) {
      const path = url.pathname.replace("/api/", "");
      const method = request.method;

      try {
        const api = new NavigationAPI(env);

        // ... 所有路由处理逻辑

        // 示例：修改返回响应的方式
        if (path === "groups" && method === "GET") {
          const groups = await api.getGroups();
          return Response.json(groups, { headers: corsHeaders });
        }

        // 为所有 Response.json 调用添加 CORS 头
        // 可以创建辅助函数
        const jsonResponse = (data: unknown, options: ResponseInit = {}) => {
          return Response.json(data, {
            ...options,
            headers: {
              ...corsHeaders,
              ...options.headers,
            },
          });
        };

        // 使用辅助函数
        if (path === "groups" && method === "GET") {
          const groups = await api.getGroups();
          return jsonResponse(groups);
        }

        // ... 其他路由

      } catch (error) {
        console.error(`API错误: ${error instanceof Error ? error.message : "未知错误"}`);
        return new Response(`处理请求时发生错误`, {
          status: 500,
          headers: corsHeaders,
        });
      }
    }

    // 非API路由默认返回404
    return new Response("Not Found", {
      status: 404,
      headers: corsHeaders,
    });
  },
} satisfies ExportedHandler;
```

**步骤 3**: 创建响应辅助函数（推荐方式）
```typescript
// worker/index.ts - 添加辅助函数

/**
 * 创建 JSON 响应并添加 CORS 头
 */
function createJsonResponse(
  data: unknown,
  request: Request,
  options: ResponseInit = {}
): Response {
  const corsHeaders = getCorsHeaders(request);

  return Response.json(data, {
    ...options,
    headers: {
      ...corsHeaders,
      ...options.headers,
    },
  });
}

// 在路由中使用
if (path === "groups" && method === "GET") {
  const groups = await api.getGroups();
  return createJsonResponse(groups, request);
}

if (path === "login" && method === "POST") {
  // ... 登录逻辑
  return createJsonResponse(result, request, {
    headers: {
      'Set-Cookie': /* ... */
    }
  });
}
```

---

### ✅ 任务 2.5: 改进错误处理和日志 [HS-005]
**优先级**: 🟠 高
**影响文件**: `worker/index.ts`
**预计时间**: 1小时

#### 修复方案

**步骤 1**: 创建错误 ID 和日志系统
```typescript
// worker/index.ts - 添加错误处理工具

/**
 * 生成唯一错误 ID
 */
function generateErrorId(): string {
  return crypto.randomUUID();
}

/**
 * 结构化日志
 */
interface LogData {
  timestamp: string;
  level: 'info' | 'warn' | 'error';
  message: string;
  errorId?: string;
  path?: string;
  method?: string;
  userId?: string;
  details?: unknown;
}

function log(data: LogData): void {
  console.log(JSON.stringify({
    ...data,
    timestamp: data.timestamp || new Date().toISOString(),
  }));
}

/**
 * 创建错误响应
 */
function createErrorResponse(
  error: unknown,
  request: Request,
  context?: string
): Response {
  const errorId = generateErrorId();
  const url = new URL(request.url);

  // 记录详细错误日志
  log({
    timestamp: new Date().toISOString(),
    level: 'error',
    message: error instanceof Error ? error.message : '未知错误',
    errorId,
    path: url.pathname,
    method: request.method,
    details: error instanceof Error ? {
      name: error.name,
      stack: error.stack,
      cause: error.cause,
    } : error,
  });

  // 返回用户友好的错误信息
  return createJsonResponse(
    {
      success: false,
      message: context ? `${context}失败` : '处理请求时发生错误',
      errorId, // 用户可以报告此 ID
    },
    request,
    { status: 500 }
  );
}
```

**步骤 2**: 在路由中应用错误处理
```typescript
// worker/index.ts - 修改主 catch 块

try {
  const api = new NavigationAPI(env);

  // ... 所有路由处理

} catch (error) {
  return createErrorResponse(error, request, 'API 请求');
}
```

**步骤 3**: 添加请求日志
```typescript
// worker/index.ts - 在 fetch 方法开始添加

async fetch(request: Request, env: Env) {
  const startTime = Date.now();
  const url = new URL(request.url);

  try {
    // ... 处理请求

    const response = /* ... */;

    // 记录成功请求
    log({
      timestamp: new Date().toISOString(),
      level: 'info',
      message: 'Request processed',
      path: url.pathname,
      method: request.method,
      details: {
        duration: Date.now() - startTime,
        status: response.status,
      },
    });

    return response;
  } catch (error) {
    return createErrorResponse(error, request);
  }
}
```

---

## 阶段 3: 中等优先级修复（2周内完成）

### ✅ 任务 3.1: 启用 TypeScript 严格模式 [MS-001]
**优先级**: 🟡 中
**影响文件**: `tsconfig.json`
**预计时间**: 3小时（包括修复类型错误）

#### 修复方案

```json
// tsconfig.json

{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,

    /* 启用严格模式 */
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "alwaysStrict": true,

    /* 额外检查 */
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,

    /* Module Resolution */
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",

    /* Interop */
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src", "worker"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

**修复类型错误**（逐步修复编译错误）

---

### ✅ 任务 3.2: 添加请求体大小限制 [MS-005]
**优先级**: 🟡 中
**影响文件**: `worker/index.ts`
**预计时间**: 30分钟

#### 修复方案

```typescript
// worker/index.ts - 添加请求体大小检查

const MAX_BODY_SIZE = 1024 * 1024; // 1MB

async function validateRequestBody(request: Request): Promise<unknown> {
  const contentLength = request.headers.get('Content-Length');

  // 检查 Content-Length 头
  if (contentLength && parseInt(contentLength, 10) > MAX_BODY_SIZE) {
    throw new Error('请求体过大');
  }

  // 读取并验证实际大小
  const bodyText = await request.text();

  if (bodyText.length > MAX_BODY_SIZE) {
    throw new Error('请求体过大');
  }

  try {
    return JSON.parse(bodyText);
  } catch {
    throw new Error('无效的 JSON 格式');
  }
}

// 在需要读取请求体的路由中使用
if (path === "login" && method === "POST") {
  try {
    const loginData = await validateRequestBody(request) as LoginInput;
    // ... 处理登录
  } catch (error) {
    return createJsonResponse(
      {
        success: false,
        message: error instanceof Error ? error.message : '请求无效',
      },
      request,
      { status: 400 }
    );
  }
}
```

---

### ✅ 任务 3.3: 添加深度数据验证 [MS-007]
**优先级**: 🟡 中
**影响文件**: `worker/index.ts`
**预计时间**: 2小时

#### 修复方案

```typescript
// worker/index.ts - 创建深度验证函数

/**
 * 深度验证导出数据
 */
function validateExportData(data: unknown): { valid: boolean; errors: string[] } {
  const errors: string[] = [];

  if (!data || typeof data !== 'object') {
    errors.push('数据必须是对象');
    return { valid: false, errors };
  }

  const d = data as any;

  // 验证 version
  if (!d.version || typeof d.version !== 'string') {
    errors.push('缺少或无效的版本信息');
  }

  // 验证 exportDate
  if (!d.exportDate || typeof d.exportDate !== 'string') {
    errors.push('缺少或无效的导出日期');
  }

  // 验证 groups
  if (!Array.isArray(d.groups)) {
    errors.push('groups 必须是数组');
  } else {
    d.groups.forEach((group: any, index: number) => {
      if (!group.name || typeof group.name !== 'string') {
        errors.push(`groups[${index}]: name 必须是字符串`);
      }
      if (typeof group.order_num !== 'number') {
        errors.push(`groups[${index}]: order_num 必须是数字`);
      }
    });
  }

  // 验证 sites
  if (!Array.isArray(d.sites)) {
    errors.push('sites 必须是数组');
  } else {
    d.sites.forEach((site: any, index: number) => {
      if (!site.name || typeof site.name !== 'string') {
        errors.push(`sites[${index}]: name 必须是字符串`);
      }
      if (!site.url || typeof site.url !== 'string') {
        errors.push(`sites[${index}]: url 必须是字符串`);
      } else {
        try {
          new URL(site.url);
        } catch {
          errors.push(`sites[${index}]: url 格式无效`);
        }
      }
      if (typeof site.group_id !== 'number') {
        errors.push(`sites[${index}]: group_id 必须是数字`);
      }
      if (typeof site.order_num !== 'number') {
        errors.push(`sites[${index}]: order_num 必须是数字`);
      }
    });
  }

  // 验证 configs
  if (!d.configs || typeof d.configs !== 'object') {
    errors.push('configs 必须是对象');
  }

  return { valid: errors.length === 0, errors };
}

// 在导入路由中使用
else if (path === "import" && method === "POST") {
  const data = await validateRequestBody(request);

  const validation = validateExportData(data);
  if (!validation.valid) {
    return createJsonResponse(
      {
        success: false,
        message: '导入数据验证失败',
        errors: validation.errors,
      },
      request,
      { status: 400 }
    );
  }

  const result = await api.importData(data as ExportData);
  return createJsonResponse(result, request);
}
```

---

### ✅ 任务 3.4: 添加 API 请求超时 [MS-010]
**优先级**: 🟡 中
**影响文件**: `src/API/client.ts`
**预计时间**: 30分钟

#### 修复方案

```typescript
// src/API/client.ts - 添加超时控制

const DEFAULT_TIMEOUT = 30000; // 30秒

export class NavigationClient {
  private baseUrl: string;
  private timeout: number;

  constructor(baseUrl: string = '/api', timeout: number = DEFAULT_TIMEOUT) {
    this.baseUrl = baseUrl;
    this.timeout = timeout;
  }

  private async request(endpoint: string, options: RequestInit = {}) {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), this.timeout);

    try {
      const response = await fetch(`${this.baseUrl}/${endpoint}`, {
        ...options,
        signal: controller.signal,
        credentials: 'include',
        headers: {
          'Content-Type': 'application/json',
          ...options.headers,
        },
      });

      clearTimeout(timeoutId);

      if (!response.ok) {
        const error = await response.text();
        throw new Error(error || `HTTP ${response.status}: ${response.statusText}`);
      }

      return response.json();
    } catch (error) {
      clearTimeout(timeoutId);

      if (error instanceof Error) {
        if (error.name === 'AbortError') {
          throw new Error('请求超时，请检查网络连接');
        }
        throw error;
      }

      throw new Error('请求失败');
    }
  }

  // 其他方法保持不变...
}
```

---

## 阶段 4: 低优先级优化（持续改进）

### ✅ 任务 4.1: 移除生产环境 console.log [LS-001]
**优先级**: 🔵 低
**预计时间**: 30分钟

```typescript
// src/utils/logger.ts - 新建

type LogLevel = 'debug' | 'info' | 'warn' | 'error';

class Logger {
  private isDevelopment: boolean;

  constructor() {
    this.isDevelopment = import.meta.env.DEV;
  }

  debug(...args: unknown[]): void {
    if (this.isDevelopment) {
      console.log('[DEBUG]', ...args);
    }
  }

  info(...args: unknown[]): void {
    console.info('[INFO]', ...args);
  }

  warn(...args: unknown[]): void {
    console.warn('[WARN]', ...args);
  }

  error(...args: unknown[]): void {
    console.error('[ERROR]', ...args);
  }
}

export const logger = new Logger();

// 使用方式
import { logger } from './utils/logger';

logger.debug('这只会在开发环境显示');
logger.error('这在所有环境都会显示');
```

---

### ✅ 任务 4.2: 提取重复代码 [LS-006]
**优先级**: 🔵 低
**预计时间**: 30分钟

```typescript
// src/utils/url.ts - 移动 extractDomain 到此处

export function extractDomain(url: string): string | null {
  // ... 实现（从 App.tsx 复制）
}

// 在 App.tsx 和 SiteSettingsModal.tsx 中导入
import { extractDomain } from '../utils/url';
```

---

### ✅ 任务 4.3: 添加无障碍标签 [LS-008]
**优先级**: 🔵 低
**预计时间**: 1小时

```typescript
// 示例：为所有 IconButton 添加 aria-label

<IconButton
  aria-label="折叠分组"
  onClick={handleToggleCollapse}
>
  <ExpandMoreIcon />
</IconButton>

<IconButton
  aria-label="编辑站点"
  onClick={handleEdit}
>
  <EditIcon />
</IconButton>
```

---

## 验证和测试计划

### 安全测试
1. **JWT 安全性测试**
   - 尝试伪造 token
   - 验证 token 过期
   - 测试签名验证

2. **XSS 测试**
   - 注入 `<script>alert(1)</script>`
   - 注入 `javascript:alert(1)`
   - 注入各种编码形式

3. **SSRF 测试**
   - 尝试 `file:///etc/passwd`
   - 尝试 `http://192.168.1.1`
   - 尝试内部端口扫描

4. **SQL 注入测试**
   - 尝试在字段中注入 SQL
   - 测试参数化查询

5. **速率限制测试**
   - 连续登录失败测试
   - 验证封禁时间

### 功能测试
- 所有现有功能仍然正常工作
- 登录/登出流程
- CRUD 操作
- 拖拽排序
- 导入/导出

### 性能测试
- 页面加载时间
- API 响应时间
- 数据库查询性能

---

## 部署清单

### 部署前准备
- [ ] 生成密码哈希值
- [ ] 创建 KV 命名空间（用于速率限制）
- [ ] 更新 wrangler.jsonc 配置
- [ ] 更新环境变量
- [ ] 运行所有测试

### 部署步骤
1. 运行 `pnpm build` 确保构建成功
2. 运行 `pnpm lint` 检查代码质量
3. 运行 `pnpm deploy` 部署到 Cloudflare
4. 验证生产环境功能
5. 监控错误日志

### 部署后验证
- [ ] 登录功能正常
- [ ] API 响应包含正确的 CORS 头
- [ ] Token 存储在 HttpOnly Cookie
- [ ] 速率限制生效
- [ ] 自定义 CSS 安全过滤
- [ ] 背景图片 URL 验证

---

## 总结

本修复方案按优先级分为 4 个阶段：

1. **阶段 1（关键）**: JWT 安全、XSS 防护、SSRF 防护、SQL 注入防护
2. **阶段 2（高优先级）**: HttpOnly Cookies、速率限制、密码哈希、CORS、错误处理
3. **阶段 3（中优先级）**: TypeScript 严格模式、请求体限制、深度验证、超时控制
4. **阶段 4（低优先级）**: 日志优化、代码重构、无障碍性

预计总工作时间：**约 20-25 小时**

建议按顺序逐个修复，每个任务完成后提交一个 commit，确保代码可追溯和可回滚。
