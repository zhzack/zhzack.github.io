---
title: "深入设计 Cloudflare Workers 跨子域统一单点登录与权限中心 (cf-auth)"
published: 2026-11-18
description: "基于 Cloudflare Workers + D1 + WebCrypto 构建的轻量级零成本统一单点登录 (SSO) 与细粒度 RBAC 权限管理中心实战设计。"
tags: [Cloudflare Workers, D1, SSO, 认证鉴权, 安全, RBAC]
category: 全栈开发
draft: false
---

在构建多个个人应用系统（如记账平台 `book.b1uegrey.xyz`、Webhook 调试台 `webhook.b1uegrey.xyz` 等）时，常常面临一个经典难题：**难道每个独立子系统都要重新设计一遍用户表、密码重置、JWT 签发与权限拦截吗？**

这不仅存在严重的重复劳动，而且各系统账号割裂，每打开一个新页面都要重新输入密码，体验极差。

为此，我设计并实现了 **`cf-auth` · 统一身份与权限管理中心 (IAM Hub)**。它完全运行在 Cloudflare Workers + D1 架构上，实现了 **一次登录、全生态子域名免密畅通、细粒度应用权限矩阵控制与银行级密码安全**。

---

## 🌟 核心目标与挑战

1. **零成本与高可用**：摒弃沉重的自建 Redis/PostgreSQL 方案，全部基于 Cloudflare 免费配额（Workers 每日 10 万次请求 + D1 每日 500 万次读）。
2. **跨二级域名单点登录 (SSO)**：在 `auth.b1uegrey.xyz` 登录后，访问 `book.b1uegrey.xyz` 或 `webhook.b1uegrey.xyz` 自动识别身份，无需弹窗重登。
3. **极速验签（<1ms）**：业务子系统不能因为鉴权产生额外的数据库往返查询，必须在边缘运行时极速判定。
4. **细粒度 RBAC 权限隔离**：支持多账号体系（管理员、普通用户、访客），并且能按应用维度（App Scope）精确控制某个用户能否进入某个子系统。

---

## 📐 跨域单点登录架构与运作机制

整个单点登录流程基于浏览器的根域名 Cookie 共享机制构建：

```
[ 访客浏览器 ]
      │
      ├────────────────────── (1) 首次访问 https://book.b1uegrey.xyz/
      │                                    │
      │ ◄──── (2) 业务网关拦截：未登录 ────────┤ (302 重定向至 auth 带 redirect 参数)
      │
      ├────────────────────── (3) 跳转到 https://auth.b1uegrey.xyz/login
      │                                    │
      │ ◄──── (4) 提交账号密码，通过校验 ─────┤ (颁发 Set-Cookie: Domain=.b1uegrey.xyz)
      │
      ├────────────────────── (5) 浏览器携 Cookie 自动跳回业务系统
      │                                    │ (Cookie 自动跨子域附带)
      │ ◄──── (6) 业务边缘节点验证 JWT 成功 ───┤ (无需查库，毫秒级直接放行)
```

### 1. 根域 Cookie 策略
当用户在 `auth.b1uegrey.xyz` 登录成功后，认证中心写入包含 JWT Token 的 Cookie：
```http
Set-Cookie: auth_token=eyJhbGciOiJIUzI1Ni...; Domain=.b1uegrey.xyz; Path=/; HttpOnly; Secure; SameSite=Lax; Max-Age=604800
```
关键点在于设置了 `Domain=.b1uegrey.xyz`。根据 RFC 6265 规范，该 Cookie 将自动被浏览器携带至所有后缀为 `.b1uegrey.xyz` 的二级与三级子域名请求中。

### 2. 子系统零查库鉴权
业务 Worker 与认证中心共享一个相同的 `JWT_SECRET`。当请求进入 `book.b1uegrey.xyz` 时：
```typescript
import { verify } from "hono/jwt";

export async function authMiddleware(c, next) {
  const token = getCookie(c, "auth_token");
  if (!token) return c.redirect(`https://auth.b1uegrey.xyz/?redirect=${encodeURIComponent(c.req.url)}`);
  
  try {
    const payload = await verify(token, c.env.JWT_SECRET);
    // 权限矩阵校验：检查当前用户是否允许访问本系统
    if (payload.role !== "admin" && !payload.allowedApps?.includes("book")) {
      return c.text("403 Forbidden: 您暂无该系统的访问权限", 403);
    }
    c.set("user", payload);
    await next();
  } catch (err) {
    return c.redirect("https://auth.b1uegrey.xyz/");
  }
}
```
验签逻辑仅在 Worker 内存中进行加密哈希计算，耗时通常 **小于 1 毫秒**，完全不需要请求任何外部服务器或数据库！

---

## 🔒 凭据安全与工程防线

### 1. 标准 WebCrypto PBKDF2 哈希存储
密码绝对不能明文或简单 MD5 存储。`cf-auth` 调用 Web 标准原生的 `crypto.subtle` API：
- 算法：**PBKDF2-HMAC-SHA256**
- 迭代次数：**100,000 次**
- 盐值：每账号生成独一无二的 16 字节硬件级强随机盐（`crypto.getRandomValues`）
- 即使数据库被意外泄露，彩虹表与 GPU 离线暴力破解也根本无法攻破。

### 2. 钓鱼重定向白名单机制
为了防止黑客构造 `auth.b1uegrey.xyz/?redirect=https://evil.com` 诱骗用户登录后跳转到钓鱼网站，系统对 `redirect` 参数做了严格的同源与二级域白名单正则过滤：
```typescript
function isValidRedirectUrl(urlStr: string, rootDomain: string): boolean {
  try {
    const parsed = new URL(urlStr);
    return parsed.protocol === "https:" && 
           (parsed.hostname === rootDomain || parsed.hostname.endsWith(`.${rootDomain}`));
  } catch {
    return false;
  }
}
```

---

## 🧭 统一应用门户导航 (Launchpad)

登录成功后，用户进入 `auth.b1uegrey.xyz` 的 Launchpad 页面，呈现九宫格仪表盘。系统会根据用户的权限矩阵动态点亮应用：
- 💰 **智能记账中心** (`book.b1uegrey.xyz`)
- 📡 **Webhook 实时调试台** (`webhook.b1uegrey.xyz`)
- 📝 **个人技术博客** (`blog.b1uegrey.xyz`)

管理员还可以在控制台随时增加新应用、分配角色权限，让未来扩展新的微服务变得轻而易举。

---

## 💡 总结

通过 `cf-auth` 的实现，我不仅彻底解决了多应用分散认证的痛点，也验证了 Cloudflare D1 + Workers 作为通用身份中枢的可靠性。一套清爽、安全且无需维护的 SSO 认证中心，正是构建个人数字化资产最坚实的护城河。
