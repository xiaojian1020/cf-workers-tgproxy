# cf-workers-tgproxy 优化说明与 Patch

本文档记录针对 `telegram_webproxy.js` 的全部优化：

## 第一轮

1. `leases` / `macKeys` 内存治理
2. `frames()` 半包缓冲
3. `mkHead()` 热路径优化
4. `mkCtr()` AES-CTR 流式 keystream

> 第一轮完整实现见 commit `365235b`。

## 第二轮

5. `timingSafeEqual()` 统一签名比较
6. 可选 `ALLOWED_HOSTS` 主机白名单
7. 分块 base64 编码

> 第二轮完整实现见 commit `1dbde38`，已通过 PR #2 合并到 `main`（merge commit `318e6ed`）。

## 1. leases / macKeys GC

### 问题

`macKeys` 与 `leases` 是模块级 `Map`，但没有主动清理。

### 修改

```js
const GC_INTERVAL = 60 * 1000;
let lastGc = 0;

function gc() {
  const now = Date.now();
  if (now - lastGc < GC_INTERVAL) return;
  lastGc = now;

  for (const [id, lease] of leases) {
    if (lease.expires < now - 5 * 60 * 1000) {
      leases.delete(id);
    }
  }

  if (macKeys.size > 1024) {
    macKeys.clear();
  }
}
```

并在 `verify()` 与 `mkSess()` 入口调用 `gc()`。

## 2. frames() 半包缓冲

### 问题

原实现在一条 WebSocket message 只包含部分帧时会直接抛错并关闭连接。

### 修改

```js
function mkFrameBuf() {
  return { buf: nil };
}

function frames(v, state = mkFrameBuf()) {
  const src = u8(v);
  state.buf = state.buf.byteLength ? cat(state.buf, src) : src;

  const out = [];
  let o = 0;

  while (o < state.buf.byteLength) {
    const part = getFrame(state.buf, o);

    // 数据不足，等待下一条 WebSocket message
    if (!part) break;

    out.push(part.frame);
    o += part.consumed;

    if (out.length >= 4096) throw Error();
  }

  state.buf = o === state.buf.byteLength ? nil : state.buf.slice(o);

  if (state.buf.byteLength > 1024 * 1024 + 8) {
    throw Error();
  }

  return out;
}
```

并在 `mkSession()` 中保存独立缓冲：

```js
const frameBuf = mkFrameBuf();
...
for (const frame of frames(b, frameBuf)) {
  ...
}
```

## 3. mkHead() 热路径优化

### 问题

原实现使用 `TextDecoder` 字符串解码和 `subarray().every()`，每次建流都会创建字符串与子数组。

### 修改

```js
function mkHead(mode, target, media) {
  const head = new Uint8Array(64);
  const bad = [
    [0x48, 0x45, 0x41, 0x44], // HEAD
    [0x50, 0x4f, 0x53, 0x54], // POST
    [0x47, 0x45, 0x54, 0x20], // GET+
    [0x4f, 0x50, 0x54, 0x49], // OPTI
  ];

  do {
    CRYPTO.getRandomValues(head);
  } while (
    head[0] === 0xef ||
    bad.some(p =>
      head[0] === p[0] &&
      head[1] === p[1] &&
      head[2] === p[2] &&
      head[3] === p[3]
    ) ||
    (head[4] === 0 && head[5] === 0 && head[6] === 0 && head[7] === 0)
  );

  head.fill(mode, 56, 60);
  new DataView(head.buffer).setInt16(
    60,
    (target + 1) * (media ? -1 : 1),
    true,
  );

  return head;
}
```

## 4. mkCtr() AES-CTR 流式 keystream

### 问题

原实现在非对齐输入时会创建对齐 `block`、复制输入、`slice()` 输出，并做复杂 counter 回退。

### 修改

```js
const mkCtr = (key, ctr) => {
  let iv = dup(ctr);
  let ks = nil;
  let ksOff = 0;

  const ready = CRYPTO.subtle.importKey(
    'raw',
    u8(key),
    { name: 'AES-CTR' },
    false,
    ['encrypt'],
  );

  const nextBlock = async () => {
    const raw = new Uint8Array(
      await CRYPTO.subtle.encrypt(
        {
          name: 'AES-CTR',
          counter: iv.slice(),
          length: 128,
        },
        await ready,
        new Uint8Array(16),
      ),
    );

    step(iv, 1);
    return raw;
  };

  const crypt = async (v) => {
    const b = u8(v);
    if (!b.byteLength) return new Uint8Array(0);

    const out = new Uint8Array(b.byteLength);
    let o = 0;

    while (o < b.byteLength) {
      if (ksOff >= ks.byteLength) {
        ks = await nextBlock();
        ksOff = 0;
      }

      const n = Math.min(16 - ksOff, b.byteLength - o);

      for (let i = 0; i < n; i++) {
        out[o + i] = b[o + i] ^ ks[ksOff + i];
      }

      o += n;
      ksOff += n;
    }

    return out;
  };

  return { crypt };
};
```

## 5. timingSafeEqual()

### 问题

原 `verify()` 中手写逐字节比较。

### 修改

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

## 6. ALLOWED_HOSTS 白名单

### 问题

原 `bridge()` 只校验 hostname 格式，不限制具体域名。

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

```txt
ALLOWED_HOSTS=your-domain.example.com
```

多个域名用英文逗号分隔；不设置时保持原有行为。

## 7. 分块 base64

### 问题

原实现 `btoa(String.fromCharCode(...b))` 在大 payload 时可能触发调用栈限制。

### 修改

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

- `frames()`：完整单帧、多帧、半包头、半 payload、超长 buffer
- `mkCtr()`：0、1、15、16、17、32 字节及跨多块非对齐输入
- `mkHead()`：不得生成 `0xef` 开头或四个坏前缀
- `timingSafeEqual()`：相同/不同长度、相同/不同内容
- `ALLOWED_HOSTS`：白名单内、白名单外、未设置
- `b64()`：0、1、32767、32768、32769 字节及更大 payload
- Telegram Web：建立 session、收发消息、关闭 lane

## 回滚

回滚第一轮：

```bash
git revert 365235b
```

回滚第二轮：

```bash
git revert 1dbde38
```