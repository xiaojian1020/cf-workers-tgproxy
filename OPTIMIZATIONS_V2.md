# cf-workers-tgproxy 第二轮优化说明与 Patch

本文档记录第二轮针对 `telegram_webproxy.js` 的 3 项优化：

1. `timingSafeEqual()` 统一签名比较
2. 可选 `ALLOWED_HOSTS` 主机白名单
3. 分块 base64 编码

> 本仓库已在 `optimize/robustness-and-crypto` 分支提交完整实现（commit `1dbde38`），并已通过 PR #2 合并到 `main`（merge commit `318e6ed`）。

## 1. timingSafeEqual()

### 问题

原 `verify()` 中手写逐字节比较：

```js
let diff = sig.byteLength ^ want.byteLength;
for (let i = 0; i < want.length; i++) diff |= sig[i] ^ want[i];
return diff === 0;
```

### 修改

新增统一工具函数：

```js
function timingSafeEqual(a, b) {
  if (a.byteLength !== b.byteLength) return false;
  let diff = 0;
  for (let i = 0; i < a.byteLength; i++) diff |= a[i] ^ b[i];
  return diff === 0;
}
```

并在 `verify()` 中替换为：

```js
return timingSafeEqual(sig, want);
```

## 2. ALLOWED_HOSTS 白名单

### 问题

原 `bridge()` 只校验 hostname 格式，不限制具体域名；任何符合格式的主机名都可能被用于生成 bridge 页面。

### 修改

`mkCfg()` 新增可选环境变量：

```js
const mkCfg = ({ SECRET, BOOT, ALLOWED_HOSTS }) => {
  ...
  const allow = String(ALLOWED_HOSTS || '')
    .split(',')
    .map(v => v.trim().toLowerCase())
    .filter(Boolean);
  return { secret, key: getKey(secret), boot, allow };
};
```

`bridge()` 增加白名单检查：

```js
async function bridge(url, secret, allow = []) {
  ...
  const host = hostOf(url.hostname);
  if (allow.length && !allow.includes(host)) return null;
  ...
}
```

路由调用改为：

```js
host = await bridge(url, cfg.secret, cfg.allow);
```

### 部署

在 Cloudflare Worker 环境变量设置：

```txt
ALLOWED_HOSTS=your-domain.example.com
```

多个域名用英文逗号分隔；不设置时保持原有行为。

## 3. 分块 base64

### 问题

原实现：

```js
btoa(String.fromCharCode(...b))
```

当 `b` 很大时，展开参数可能触发调用栈限制。

### 修改

改为按 32 KiB 分块：

```js
function b64(v) {
  const b = v instanceof Uint8Array ? v : new Uint8Array(v);
  let s = '';
  for (let i = 0; i < b.byteLength; i += 0x8000) {
    s += String.fromCharCode(...b.subarray(i, i + 0x8000));
  }
  return btoa(s)
    .replaceAll('+', '-')
    .replaceAll('/', '_')
    .replace(/=+$/, '');
}
```

## 回归测试

- `timingSafeEqual()`：相同/不同长度、相同/不同内容
- `ALLOWED_HOSTS`：白名单内、白名单外、未设置
- `b64()`：0、1、32767、32768、32769 字节及更大 payload
- Telegram Web：建立 session、收发消息、关闭 lane

## 回滚

如需回滚第二轮代码变更，revert 对应 commit：

```bash
git revert 1dbde38
```

或回退到上一轮合并点：

```bash
git checkout 9a1bd3b
```