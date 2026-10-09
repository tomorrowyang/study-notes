# 前端安全手写题

> 目标：能手写常见的防御代码，并说清每一处边界条件为什么这样处理。所有题目只涉及防御实现。

## 使用方式

- 输出与清洗：H1 上下文转义、H2 安全的 URL 与跳转、H3 白名单 HTML 清洗、H11 Trusted Types 策略
- 请求与认证：H4 CSP nonce 中间件、H5 签名的双重提交 CSRF 令牌、H6 PKCE、H9 JWT 校验、H12 多标签页登出同步
- 跨源与输入：H7 安全的 postMessage、H8 防原型污染的深合并、H10 防 SSRF 的请求

先自己写，再对照参考实现和易错点。

---

### H1. 按上下文转义输出

- **难度**: 初级
- **标签**: XSS, 转义
- **面试官看什么**: 是否知道不同上下文需要不同的转义

**题干**

实现：

- `escapeHtml(str)`：用于 HTML 文本和**带引号的**属性值
- `serializeForScript(data)`：把数据安全地嵌入 `<script>` 标签中（SSR 注入初始状态）

**参考实现**

```ts
const HTML_ESCAPES: Record<string, string> = {
  '&': '&amp;',
  '<': '&lt;',
  '>': '&gt;',
  '"': '&quot;',
  "'": '&#39;',
};

/**
 * 转义 HTML 文本和带引号的属性值。
 * 不适用于：未加引号的属性、URL 属性（需要额外校验协议）、script/style 内容。
 */
export function escapeHtml(input: string): string {
  return input.replace(/[&<>"']/g, (ch) => HTML_ESCAPES[ch]);
}

/**
 * 把数据序列化为可以放在 <script> 标签内的 JSON。
 */
export function serializeForScript(data: unknown): string {
  return (
    JSON.stringify(data)
      // JSON.stringify 不转义 <，数据中的 "</script>" 会提前结束标签
      .replace(/</g, '\\u003c')
      .replace(/>/g, '\\u003e')
      .replace(/&/g, '\\u0026')
      // U+2028/2029 在旧的 JS 引擎中是非法的字符串字符
      .replace(/\u2028/g, '\\u2028')
      .replace(/\u2029/g, '\\u2029')
  );
}

// 使用：
// `<div title="${escapeHtml(title)}">${escapeHtml(content)}</div>`
// `<script>window.__STATE__ = ${serializeForScript(state)}</script>`
```

**易错点**

- `&` 必须转义，否则已有的实体会被误解析
- 单引号也要转义（属性可能用单引号包裹）
- 嵌入 `<script>` 时用 `escapeHtml` 是错的（script 内容不解析实体）；用 `JSON.stringify` 也不够（没有处理 `</script>`）
- `href` 中的 `javascript:` 不会被 HTML 转义阻止 → 见 H2

---

### H2. 安全的 URL 与登录后跳转

- **难度**: 中级
- **标签**: XSS, 开放重定向
- **面试官看什么**: 是否用 URL 解析器而不是字符串匹配

**题干**

1. `toSafeHref(url)`：用户提供的链接只允许 http、https、mailto，否则返回 `'about:blank'`
2. `getSafeRedirect(target, fallback)`：登录后跳转，只允许同源的路径

**参考实现**

```ts
const ALLOWED_PROTOCOLS = new Set(['http:', 'https:', 'mailto:']);

/**
 * 把用户提供的链接转为可以放入 href 的安全值。
 */
export function toSafeHref(url: string): string {
  try {
    // 用 URL 解析器而不是正则：浏览器会忽略协议中的空白、制表符和大小写
    // （例如 " JaVa\tScript:"），字符串匹配很难覆盖所有变体
    const parsed = new URL(url.trim(), window.location.origin);
    return ALLOWED_PROTOCOLS.has(parsed.protocol) ? parsed.href : 'about:blank';
  } catch {
    return 'about:blank';
  }
}

/**
 * 校验登录后的跳转目标，只允许同源的路径，否则返回 fallback。
 */
export function getSafeRedirect(target: string | null, fallback = '/'): string {
  if (!target) return fallback;

  // 必须以单个 / 开头：
  // "//evil.com" 是协议相对 URL，"/\evil.com" 在浏览器中会把 \ 规范化为 /
  if (!target.startsWith('/') || target.startsWith('//') || target.startsWith('/\\')) {
    return fallback;
  }

  try {
    const base = window.location.origin;
    const parsed = new URL(target, base);
    // 最终以解析后的源为准，覆盖其他未预料到的变体
    if (parsed.origin !== base) return fallback;
    return parsed.pathname + parsed.search + parsed.hash;
  } catch {
    return fallback;
  }
}
```

**易错点**

- 用 `startsWith('http')` 或 `includes('example.com')` 判断（`https://example.com.evil.com`）
- 只检查以 `/` 开头，漏掉 `//evil.com`
- 返回原始输入而不是解析后的值（解析前后可能不一致）
- React 19 会阻止 `javascript:` URL，但其他协议（`data:`）和非 React 的场景仍然需要处理

---

### H3. 基于白名单的 HTML 清洗器

- **难度**: 高级
- **标签**: XSS, 富文本
- **面试官看什么**: 白名单思路、用浏览器解析器、属性和 URL 的处理

**题干**

实现简化的 `sanitize(html)`：只保留白名单中的标签和属性，链接只允许安全的协议。生产环境请使用 DOMPurify，这道题考察原理。

**参考实现**

```ts
const ALLOWED_TAGS = new Set(['P', 'BR', 'B', 'I', 'EM', 'STRONG', 'UL', 'OL', 'LI', 'A', 'BLOCKQUOTE', 'CODE', 'PRE']);
const ALLOWED_ATTRS: Record<string, Set<string>> = {
  A: new Set(['href', 'title']),
};
// 这些标签的内容本身也不应该保留（不是只去掉标签留下文本）
const DROP_WITH_CONTENT = new Set(['SCRIPT', 'STYLE', 'TEMPLATE', 'IFRAME', 'OBJECT', 'EMBED', 'NOSCRIPT']);

/**
 * 简化版白名单 HTML 清洗器（学习用，生产请使用 DOMPurify）。
 */
export function sanitize(html: string): string {
  // DOMParser 解析的文档是惰性的：不执行脚本，也不加载图片（不会触发 onerror）
  const doc = new DOMParser().parseFromString(html, 'text/html');
  cleanChildren(doc.body);
  return doc.body.innerHTML;
}

function cleanChildren(parent: Element): void {
  // 先复制为数组：遍历过程中会删除和替换节点
  for (const node of Array.from(parent.childNodes)) {
    if (node.nodeType === Node.TEXT_NODE) continue;

    if (node.nodeType !== Node.ELEMENT_NODE) {
      node.remove(); // 注释等其他节点
      continue;
    }

    const el = node as Element;
    if (DROP_WITH_CONTENT.has(el.tagName)) {
      el.remove();
      continue;
    }

    if (!ALLOWED_TAGS.has(el.tagName)) {
      // 不在白名单中的标签：去掉标签本身，保留清洗后的子内容
      cleanChildren(el);
      el.replaceWith(...Array.from(el.childNodes));
      continue;
    }

    cleanAttributes(el);
    cleanChildren(el);
  }
}

function cleanAttributes(el: Element): void {
  const allowed = ALLOWED_ATTRS[el.tagName] ?? new Set<string>();
  for (const attr of Array.from(el.attributes)) {
    if (!allowed.has(attr.name)) {
      el.removeAttribute(attr.name);
      continue;
    }
    if (attr.name === 'href' && !isSafeLink(attr.value)) {
      el.removeAttribute('href');
    }
  }
  if (el.tagName === 'A' && el.hasAttribute('href')) {
    // 用户链接在新标签打开时防止反向控制，并不传递权重
    el.setAttribute('rel', 'noopener noreferrer nofollow');
  }
}

function isSafeLink(value: string): boolean {
  try {
    const url = new URL(value, 'https://placeholder.invalid');
    return ['http:', 'https:', 'mailto:'].includes(url.protocol);
  } catch {
    return false;
  }
}
```

**易错点**

- 用黑名单（删除 script 和 on* 属性）→ 总有遗漏
- 用 `innerHTML` 赋值到活动文档中的元素来解析 → `<img onerror>` 会立即执行
- 清洗后再修改 HTML（拼接、二次解析）→ 可能出现变异 XSS
- 不处理 `href` 的协议
- 遍历 `childNodes` 时直接删除（实时集合导致跳过节点）

---

### H4. CSP nonce 中间件

- **难度**: 中级
- **标签**: CSP
- **面试官看什么**: nonce 的生成、每次请求唯一、注入方式

**题干**

为 Express 风格的 SSR 服务实现中间件：每个请求生成 nonce，设置严格的 CSP 头，让模板可以使用 nonce。

**参考实现**

```ts
import { randomBytes } from 'node:crypto';
import type { Request, Response, NextFunction } from 'express';

interface CspOptions {
  reportOnly?: boolean;
  reportEndpoint?: string;
}

/**
 * 为每个请求生成 CSP nonce，并设置基于 nonce + strict-dynamic 的 CSP 头。
 * nonce 放在 res.locals.cspNonce，供模板渲染 <script nonce>。
 */
export function cspNonce(options: CspOptions = {}) {
  return (req: Request, res: Response, next: NextFunction) => {
    // 必须每个请求重新生成且不可预测；固定的 nonce 等同于没有
    const nonce = randomBytes(16).toString('base64');
    res.locals.cspNonce = nonce;

    const directives = [
      // 'unsafe-inline' 和 https: 只用于不支持 nonce / strict-dynamic 的旧浏览器；
      // 支持的浏览器会忽略它们
      `script-src 'nonce-${nonce}' 'strict-dynamic' https: 'unsafe-inline'`,
      `object-src 'none'`,
      `base-uri 'none'`,
      `frame-ancestors 'self'`,
    ];
    if (options.reportEndpoint) {
      res.setHeader('Reporting-Endpoints', `csp="${options.reportEndpoint}"`);
      directives.push('report-to csp');
    }

    const header = options.reportOnly
      ? 'Content-Security-Policy-Report-Only'
      : 'Content-Security-Policy';
    res.setHeader(header, directives.join('; '));

    // 带 nonce 的 HTML 不能被共享缓存复用，否则所有用户拿到同一个 nonce
    res.setHeader('Cache-Control', 'private, no-store');
    next();
  };
}

// 模板中：<script nonce="<%= cspNonce %>" src="/app.js"></script>
```

**易错点**

- nonce 用 `Math.random()` 或固定值
- 带 nonce 的页面被 CDN 缓存
- 把 nonce 拼接到用户可控的内容中（例如把 nonce 加到所有 script 标签，包括注入的）
- 忘记 `base-uri`（攻击者注入 `<base>` 改变相对路径脚本的来源）

---

### H5. 签名的双重提交 CSRF 令牌

- **难度**: 中级
- **标签**: CSRF
- **面试官看什么**: 为什么要签名、绑定会话、常量时间比较

**题干**

实现：

- `issueCsrfToken(sessionId)`：生成绑定会话的 CSRF 令牌
- `verifyCsrfToken(sessionId, token)`：校验

令牌通过 Cookie（非 HttpOnly，前端读取）下发，前端在请求头 `X-CSRF-Token` 中提交。

**参考实现**

```ts
import { createHmac, randomBytes, timingSafeEqual } from 'node:crypto';

const SECRET = process.env.CSRF_SECRET!;

function sign(sessionId: string, random: string): string {
  // 签名包含 sessionId：即使攻击者能在子域名设置 Cookie（Cookie 注入），
  // 也无法伪造与受害者会话匹配的令牌
  return createHmac('sha256', SECRET).update(`${sessionId}!${random}`).digest('base64url');
}

/**
 * 生成绑定会话的 CSRF 令牌，格式为 random.signature。
 */
export function issueCsrfToken(sessionId: string): string {
  const random = randomBytes(16).toString('base64url');
  return `${random}.${sign(sessionId, random)}`;
}

/**
 * 校验请求头中的 CSRF 令牌是否属于当前会话。
 */
export function verifyCsrfToken(sessionId: string, token: string | undefined): boolean {
  if (!token) return false;
  const [random, signature] = token.split('.');
  if (!random || !signature) return false;

  const expected = Buffer.from(sign(sessionId, random));
  const actual = Buffer.from(signature);
  // timingSafeEqual 要求长度相同，长度不同直接判定失败
  return expected.length === actual.length && timingSafeEqual(expected, actual);
}

// 中间件中：对 POST/PUT/PATCH/DELETE 请求，
// verifyCsrfToken(req.session.id, req.get('X-CSRF-Token')) 不通过则返回 403
```

**易错点**

- 不签名的双重提交（只比较 Cookie 和请求头）→ 子域名可以注入 Cookie
- 令牌不绑定会话
- 用 `===` 比较签名（时序攻击）
- 对 GET 请求要求令牌却允许 GET 修改数据（应该让 GET 无副作用）
- 登录后不重新生成令牌（会话 ID 变了）

---

### H6. PKCE 参数生成

- **难度**: 中级
- **标签**: OAuth
- **面试官看什么**: Web Crypto、base64url、state 的作用

**题干**

在浏览器中生成 OAuth 授权码流程所需的 `code_verifier`、`code_challenge`（S256）和 `state`，并构造授权 URL。

**参考实现**

```ts
function base64url(bytes: Uint8Array): string {
  let binary = '';
  for (const b of bytes) binary += String.fromCharCode(b);
  return btoa(binary).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
}

function randomString(byteLength = 32): string {
  return base64url(crypto.getRandomValues(new Uint8Array(byteLength)));
}

/**
 * 生成 PKCE 的 verifier 和 S256 challenge。
 */
export async function createPkcePair(): Promise<{ verifier: string; challenge: string }> {
  // 32 字节随机数编码后是 43 个字符，满足规范要求的 43–128 长度
  const verifier = randomString(32);
  const digest = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(verifier));
  return { verifier, challenge: base64url(new Uint8Array(digest)) };
}

interface AuthorizeParams {
  authorizeEndpoint: string;
  clientId: string;
  redirectUri: string;
  scope: string;
}

/**
 * 构造授权 URL，并把 verifier 和 state 存入 sessionStorage 供回调使用。
 */
export async function buildAuthorizeUrl(params: AuthorizeParams): Promise<string> {
  const { verifier, challenge } = await createPkcePair();
  const state = randomString(16);
  // sessionStorage 只在当前标签页有效，回调结束后立即删除
  sessionStorage.setItem('oauth_pkce', JSON.stringify({ verifier, state }));

  const url = new URL(params.authorizeEndpoint);
  url.search = new URLSearchParams({
    response_type: 'code',
    client_id: params.clientId,
    redirect_uri: params.redirectUri,
    scope: params.scope,
    state,
    code_challenge: challenge,
    code_challenge_method: 'S256',
  }).toString();
  return url.toString();
}

/**
 * 回调页校验 state，返回用于换取令牌的 verifier。
 */
export function consumeCallback(returnedState: string | null): string {
  const raw = sessionStorage.getItem('oauth_pkce');
  sessionStorage.removeItem('oauth_pkce'); // 一次性使用，无论成功与否
  if (!raw) throw new Error('缺少 PKCE 上下文');
  const { verifier, state } = JSON.parse(raw) as { verifier: string; state: string };
  // state 不匹配说明回调不是本次发起的授权（登录 CSRF）
  if (!returnedState || returnedState !== state) throw new Error('state 不匹配');
  return verifier;
}
```

**易错点**

- 用 `Math.random()` 生成 verifier
- 用普通 base64（含 `+/=`）
- 使用 `plain` 方法
- 不校验 state，或校验后不删除
- 有 BFF 时，这些逻辑应该放在服务端（令牌不进浏览器）

---

### H7. 安全的 postMessage 通信

- **难度**: 中级
- **标签**: postMessage
- **面试官看什么**: 校验 origin 和 source、指定 targetOrigin、校验数据结构

**题干**

实现父页面与嵌入的跨源 iframe 之间的请求/响应通信封装。

**参考实现**

```ts
interface RpcRequest {
  type: 'rpc-request';
  id: string;
  method: string;
  params: unknown;
}

interface RpcResponse {
  type: 'rpc-response';
  id: string;
  result?: unknown;
  error?: string;
}

function isRpcResponse(data: unknown): data is RpcResponse {
  return (
    typeof data === 'object' &&
    data !== null &&
    (data as RpcResponse).type === 'rpc-response' &&
    typeof (data as RpcResponse).id === 'string'
  );
}

/**
 * 父页面调用 iframe 中方法的 RPC 客户端。
 */
export class FrameRpcClient {
  private pending = new Map<string, { resolve: (v: unknown) => void; reject: (e: Error) => void }>();

  constructor(
    private readonly frame: HTMLIFrameElement,
    private readonly frameOrigin: string, // 例如 'https://widget.example.com'
  ) {
    window.addEventListener('message', this.onMessage);
  }

  call(method: string, params: unknown, timeoutMs = 5000): Promise<unknown> {
    const id = crypto.randomUUID();
    const message: RpcRequest = { type: 'rpc-request', id, method, params };

    return new Promise((resolve, reject) => {
      const timer = setTimeout(() => {
        this.pending.delete(id);
        reject(new Error(`调用 ${method} 超时`));
      }, timeoutMs);

      this.pending.set(id, {
        resolve: (v) => { clearTimeout(timer); resolve(v); },
        reject: (e) => { clearTimeout(timer); reject(e); },
      });

      // 指定 targetOrigin：如果 iframe 被导航到其他源，消息不会被投递
      this.frame.contentWindow?.postMessage(message, this.frameOrigin);
    });
  }

  private onMessage = (event: MessageEvent) => {
    // 同时校验 origin 和 source：同一个源可能有多个窗口
    if (event.origin !== this.frameOrigin) return;
    if (event.source !== this.frame.contentWindow) return;
    if (!isRpcResponse(event.data)) return;

    const entry = this.pending.get(event.data.id);
    if (!entry) return;
    this.pending.delete(event.data.id);
    if (event.data.error) entry.reject(new Error(event.data.error));
    else entry.resolve(event.data.result);
  };

  destroy(): void {
    window.removeEventListener('message', this.onMessage);
    for (const entry of this.pending.values()) entry.reject(new Error('已销毁'));
    this.pending.clear();
  }
}
```

**易错点**

- 使用 `'*'` 作为 targetOrigin
- 不校验 `event.origin`，或用 `includes`、`endsWith` 校验
- 把收到的数据直接用于 `innerHTML`、`location.href`、`eval`
- 不清理事件监听和挂起的请求
- iframe 侧同样需要校验来源（双向）

---

### H8. 防原型污染的深合并

- **难度**: 中级
- **标签**: 原型污染
- **面试官看什么**: 危险键、只合并普通对象、自有属性

**题干**

实现 `safeDeepMerge(target, source)`，`source` 来自用户输入（JSON）。

**参考实现**

```ts
const FORBIDDEN_KEYS = new Set(['__proto__', 'constructor', 'prototype']);

function isPlainObject(value: unknown): value is Record<string, unknown> {
  if (typeof value !== 'object' || value === null) return false;
  const proto = Object.getPrototypeOf(value);
  return proto === Object.prototype || proto === null;
}

/**
 * 深合并普通对象，跳过可能导致原型污染的键。返回新对象，不修改参数。
 */
export function safeDeepMerge<T extends Record<string, unknown>>(
  target: T,
  source: unknown,
): T {
  // 结果使用无原型对象，即使出现遗漏的键也不会写到 Object.prototype 上
  const result: Record<string, unknown> = Object.assign(Object.create(null), target);
  if (!isPlainObject(source)) return result as T;

  // Object.keys 只返回自有的可枚举键。
  // 注意 JSON.parse('{"__proto__":{}}') 会产生一个名为 "__proto__" 的自有属性，所以仍需过滤
  for (const key of Object.keys(source)) {
    if (FORBIDDEN_KEYS.has(key)) continue;

    const srcVal = source[key];
    const tgtVal = result[key];
    result[key] =
      isPlainObject(srcVal) && isPlainObject(tgtVal)
        ? safeDeepMerge(tgtVal, srcVal)
        : isPlainObject(srcVal)
          ? safeDeepMerge({}, srcVal)
          : srcVal;
  }
  return result as T;
}

// 读取配置时也用自有属性判断：Object.hasOwn(config, 'isAdmin')
```

**易错点**

- 只过滤 `__proto__`，漏掉 `constructor.prototype`
- 用 `for...in`（会遍历继承的属性）
- 直接修改 target
- 没有意识到 `JSON.parse` 会产生 `__proto__` 自有属性
- 返回的无原型对象没有 `toString` 等方法；如果调用方需要普通对象，在边界处转换

---

### H9. 校验 JWT（HS256）

- **难度**: 高级
- **标签**: JWT
- **面试官看什么**: 固定算法、签名校验、声明校验、常见的绕过点

**题干**

在 Edge / 浏览器环境（Web Crypto）中实现 HS256 JWT 的校验。生产环境请使用成熟的库（jose），这道题考察原理。

**参考实现**

```ts
interface JwtClaims {
  sub: string;
  iss: string;
  aud: string | string[];
  exp: number;
  nbf?: number;
  [key: string]: unknown;
}

interface VerifyOptions {
  secret: string;
  issuer: string;
  audience: string;
  clockToleranceSec?: number;
}

function base64urlDecode(input: string): Uint8Array {
  const base64 = input.replace(/-/g, '+').replace(/_/g, '/');
  const padded = base64 + '='.repeat((4 - (base64.length % 4)) % 4);
  return Uint8Array.from(atob(padded), (c) => c.charCodeAt(0));
}

function decodeJson<T>(segment: string): T {
  return JSON.parse(new TextDecoder().decode(base64urlDecode(segment))) as T;
}

/**
 * 校验 HS256 JWT 的签名和标准声明，失败时抛出错误。
 */
export async function verifyJwtHS256(token: string, options: VerifyOptions): Promise<JwtClaims> {
  const parts = token.split('.');
  if (parts.length !== 3) throw new Error('格式错误');
  const [headerSeg, payloadSeg, signatureSeg] = parts;

  const header = decodeJson<{ alg?: string; typ?: string }>(headerSeg);
  // 算法由服务端固定，绝不能根据令牌头部选择：
  // 否则攻击者可以把 alg 改成 none，或在 RS256 / HS256 之间混淆
  if (header.alg !== 'HS256') throw new Error('不支持的算法');

  const key = await crypto.subtle.importKey(
    'raw',
    new TextEncoder().encode(options.secret),
    { name: 'HMAC', hash: 'SHA-256' },
    false,
    ['verify'],
  );
  // subtle.verify 内部使用常量时间比较
  const valid = await crypto.subtle.verify(
    'HMAC',
    key,
    base64urlDecode(signatureSeg),
    new TextEncoder().encode(`${headerSeg}.${payloadSeg}`),
  );
  if (!valid) throw new Error('签名无效');

  // 签名通过后才信任载荷
  const claims = decodeJson<JwtClaims>(payloadSeg);
  const now = Math.floor(Date.now() / 1000);
  const tolerance = options.clockToleranceSec ?? 30;

  if (typeof claims.exp !== 'number' || now > claims.exp + tolerance) throw new Error('已过期');
  if (typeof claims.nbf === 'number' && now + tolerance < claims.nbf) throw new Error('尚未生效');
  if (claims.iss !== options.issuer) throw new Error('签发者不匹配');
  const audiences = Array.isArray(claims.aud) ? claims.aud : [claims.aud];
  // 校验受众：防止把签发给其他服务的令牌拿来使用
  if (!audiences.includes(options.audience)) throw new Error('受众不匹配');

  return claims;
}
```

**易错点**

- 信任头部的 `alg`（`none`、算法混淆）
- 先解析载荷并使用，再校验签名
- 不校验 `exp`（或缺少 `exp` 时放行）、`aud`、`iss`
- 自己用 `===` 比较签名字符串
- 弱密钥（HS256 的密钥应该是足够长的随机值）
- 前端解码 JWT 读取信息可以，但**前端的校验不能作为授权依据**

---

### H10. 防 SSRF 的服务端请求

- **难度**: 高级
- **标签**: SSRF
- **面试官看什么**: DNS 解析后检查、固定 IP、重定向逐跳检查

**题干**

实现 Node.js 的 `safeFetch(url)`，用于「抓取用户提供的 URL 生成链接预览」。要求阻止访问内网、云元数据、本地回环地址。

**参考实现**

```ts
import { lookup } from 'node:dns/promises';
import { isIP, BlockList } from 'node:net';
// 使用 undici 自带的 fetch，保证与 Agent 版本一致
import { Agent, fetch, type Response } from 'undici';

const blockList = new BlockList();
// 私有、回环、链路本地（含云元数据 169.254.169.254）、CGNAT、保留地址
for (const [net, prefix] of [
  ['0.0.0.0', 8], ['10.0.0.0', 8], ['100.64.0.0', 10], ['127.0.0.0', 8],
  ['169.254.0.0', 16], ['172.16.0.0', 12], ['192.168.0.0', 16], ['224.0.0.0', 4], ['240.0.0.0', 4],
] as const) {
  blockList.addSubnet(net, prefix, 'ipv4');
}
for (const [net, prefix] of [
  ['::', 128], ['::1', 128], ['fc00::', 7], ['fe80::', 10], ['ff00::', 8],
] as const) {
  blockList.addSubnet(net, prefix, 'ipv6');
}

function isBlockedAddress(address: string): boolean {
  // IPv4 映射的 IPv6 地址（::ffff:127.0.0.1）要按 IPv4 检查
  const mapped = address.match(/^::ffff:(\d+\.\d+\.\d+\.\d+)$/i);
  if (mapped) return blockList.check(mapped[1], 'ipv4');
  return blockList.check(address, isIP(address) === 6 ? 'ipv6' : 'ipv4');
}

/**
 * 解析域名并确认所有地址都是公网地址，返回用于连接的地址。
 */
async function resolvePublicAddress(hostname: string): Promise<{ address: string; family: number }> {
  const records = isIP(hostname)
    ? [{ address: hostname, family: isIP(hostname) }]
    : await lookup(hostname, { all: true });
  if (records.length === 0) throw new Error('无法解析');
  // 任一地址是内网就拒绝：否则攻击者可以返回「公网 + 内网」的混合记录
  if (records.some((r) => isBlockedAddress(r.address))) throw new Error('目标地址不允许');
  return records[0];
}

/**
 * 抓取用户提供的公网 URL，逐跳校验重定向，连接时固定使用已校验的 IP。
 */
export async function safeFetch(rawUrl: string, maxRedirects = 3): Promise<Response> {
  let url = new URL(rawUrl);

  for (let hop = 0; hop <= maxRedirects; hop++) {
    if (url.protocol !== 'http:' && url.protocol !== 'https:') throw new Error('协议不允许');
    if (url.username || url.password) throw new Error('不允许携带凭证');

    const { address, family } = await resolvePublicAddress(url.hostname.replace(/^\[|\]$/g, ''));

    // 固定连接到已校验的 IP，防止 DNS 重绑定：
    // 如果连接时再次解析，第二次可能返回内网地址。
    // 仍然使用原始的主机名，保证 TLS 的 SNI 和证书校验正确
    const dispatcher = new Agent({
      connect: {
        // Node 开启 autoSelectFamily 时会以 all: true 调用 lookup，两种回调格式都要支持
        lookup: (_host, opts, cb) =>
          opts.all ? cb(null, [{ address, family }]) : cb(null, address, family),
      },
    });

    const res = await fetch(url, {
      redirect: 'manual', // 自动跟随重定向会绕过对下一跳的检查
      signal: AbortSignal.timeout(5000),
      dispatcher,
    });

    if (res.status >= 300 && res.status < 400) {
      const location = res.headers.get('location');
      if (!location) return res;
      url = new URL(location, url);
      continue;
    }
    return res;
  }
  throw new Error('重定向次数过多');
}
```

**易错点**

- 只检查 URL 中的主机名字符串（`127.1`、十进制 IP、指向内网的域名都能绕过）
- 解析检查后再让 fetch 重新解析（DNS 重绑定）
- 自动跟随重定向
- 漏掉 IPv6 和 IPv4 映射地址
- 不限制响应大小和超时
- 更稳妥的做法是**网络层隔离**：抓取服务部署在无法访问内网的网络环境中，代码检查作为第二层

---

### H11. Trusted Types 策略

- **难度**: 高级
- **标签**: Trusted Types, XSS
- **面试官看什么**: 策略的集中管理、兼容未支持的浏览器

**题干**

为应用创建 Trusted Types 策略：HTML 必须经过 DOMPurify 清洗，脚本 URL 只允许自己的 CDN；在不支持 Trusted Types 的浏览器中也能正常工作。

**参考实现**

```ts
import DOMPurify from 'dompurify';

const SCRIPT_ORIGINS = new Set(['https://static.example.com']);

interface AppPolicy {
  createHTML(input: string): string | TrustedHTML;
  createScriptURL(input: string): string | TrustedScriptURL;
}

function createPolicy(): AppPolicy {
  const rules = {
    createHTML: (input: string) => DOMPurify.sanitize(input),
    createScriptURL: (input: string) => {
      const url = new URL(input, window.location.origin);
      if (url.origin === window.location.origin || SCRIPT_ORIGINS.has(url.origin)) {
        return url.href;
      }
      throw new TypeError(`不允许的脚本地址：${url.origin}`);
    },
  };

  // 不支持 Trusted Types 的浏览器回退为普通函数：同样会清洗和校验，只是没有浏览器强制
  if (typeof window === 'undefined' || !window.trustedTypes) return rules;

  // 策略名称需要在 CSP 的 trusted-types 指令中声明，同名策略只能创建一次
  return window.trustedTypes.createPolicy('app', rules);
}

/**
 * 应用唯一的 Trusted Types 策略，所有危险汇的写入都必须经过它。
 */
export const appPolicy = createPolicy();

/**
 * 安全地设置元素的 HTML 内容。
 */
export function setSafeHTML(el: Element, html: string): void {
  // TrustedHTML 可以直接赋值给 innerHTML；类型声明里 innerHTML 是 string，因此需要断言
  el.innerHTML = appPolicy.createHTML(html) as string;
}

// 对应的 CSP：
// require-trusted-types-for 'script'; trusted-types app dompurify
```

**易错点**

- 策略中直接返回输入（「放行策略」），失去意义
- 在很多地方创建多个策略 → 难以审查；CSP 中限制允许的策略名称
- 设置 `default` 策略时把所有内容都放行
- 忘记第三方库可能需要自己的策略名（如 `dompurify`）
- 先用 Report-Only 收集违规再强制执行

---

### H12. 多标签页的登出同步

- **难度**: 中级
- **标签**: 会话
- **面试官看什么**: BroadcastChannel、不广播敏感数据、服务端失效

**题干**

用户在一个标签页登出后，其他标签页应该立即清理本地数据并跳转到登录页；会话被服务端撤销时（401）也一样。

**参考实现**

```ts
type SessionEvent = { type: 'logout'; reason: 'user' | 'expired' };

const channel = typeof BroadcastChannel !== 'undefined'
  ? new BroadcastChannel('session')
  : null;

/**
 * 清理当前标签页中的敏感数据并跳转到登录页。
 */
function clearLocalSession(reason: SessionEvent['reason']): void {
  // 内存中的令牌、缓存的用户数据、可能包含个人信息的存储都要清理
  sessionStorage.clear();
  localStorage.removeItem('draft');
  // 用 replace：防止通过后退回到已登录的页面
  window.location.replace(`/login?reason=${reason}`);
}

/**
 * 登出：先让服务端会话失效，再通知其他标签页。
 */
export async function logout(): Promise<void> {
  try {
    // 必须让服务端会话失效；只清理前端数据时，被窃取的 Cookie 仍然有效
    await fetch('/api/auth/logout', { method: 'POST', credentials: 'include' });
  } finally {
    // 只广播事件，不广播令牌等敏感数据
    channel?.postMessage({ type: 'logout', reason: 'user' } satisfies SessionEvent);
    clearLocalSession('user');
  }
}

/**
 * 请求返回 401 时调用，处理服务端撤销的会话。
 */
export function handleUnauthorized(): void {
  channel?.postMessage({ type: 'logout', reason: 'expired' } satisfies SessionEvent);
  clearLocalSession('expired');
}

channel?.addEventListener('message', (event: MessageEvent<SessionEvent>) => {
  if (event.data?.type === 'logout') clearLocalSession(event.data.reason);
});

// 页面从 bfcache 恢复时，可能显示已登出用户的旧页面
window.addEventListener('pageshow', (event) => {
  if (event.persisted) {
    fetch('/api/auth/session', { credentials: 'include' }).then((res) => {
      if (res.status === 401) clearLocalSession('expired');
    });
  }
});
```

**易错点**

- 只在前端清理数据，服务端会话仍然有效
- 通过 BroadcastChannel 或 localStorage 广播令牌
- 用 `location.href` 跳转（可以后退回来）
- 忽略 bfcache 恢复的页面
- 登出接口使用 GET（可以被跨站触发登出）
