# cf-workers-tgproxy 优化说明与 Patch

本文档记录针对 `telegram_webproxy.js` 的 4 项优化：

1. `leases` / `macKeys` 内存治理
2. `frames()` 半包缓冲
3. `mkHead()` 热路径优化
4. `mkCtr()` AES-CTR 流式 keystream

> 注意：本仓库已在 `optimize/hotpath-frame-buffer` 分支提交了完整实现（commit `365235b`）。本文档用于说明改动、便于 review 与回滚。

## 1. leases / macKeys GC

### 问题

`macKeys` 与 `leases` 是模块级 `Map`，但没有主动清理：

- 每个 `BOOT`、每个 bridge secret 都可能在 `macKeys` 留下 `CryptoKey`
- session 过期后，`leases` 仍可能保留记录

### 修改

新增：

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

并在以下入口调用：

```js
async function verify(v, boot, kind) {
  gc();
  ...
}

const mkSess = async boot => {
  gc();
  return mkToken(boot, 'session', Date.now() + 5 * 60 * 1000, rand(12));
};
```

## 2. frames() 半包缓冲

### 问题

原实现：

```js
function frames(v) {
  ...
  const part = getFrame(src, o);
  if (!part) throw Error();
  ...
}
```

当一条 WebSocket message 只包含部分帧时，会直接抛错并关闭连接。

### 修改

改为有状态解析：

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
const mkSession = ({ secret, sid, onClose, lease }) => {
  ...
  const frameBuf = mkFrameBuf();

  const onData = (v) => {
    ...
    for (const frame of frames(b, frameBuf)) {
      ...
    }
  };
};
```

## 3. mkHead() 热路径优化

### 问题

原实现使用：

```js
badHead.has(dec.decode(head.subarray(0, 4)))
head.subarray(4, 8).every(v => v === 0)
```

每次建流都会创建字符串与子数组。

### 修改

改为字节级比较：

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

原实现在非对齐输入时会：

- 创建对齐 `block`
- 复制输入
- `slice()` 输出
- 复杂回退 counter

### 修改

改为维护 16 字节 keystream：

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

## 回归测试

建议至少覆盖：

- `frames()`：完整单帧、多帧、半包头、半 payload、超长 buffer
- `mkCtr()`：0、1、15、16、17、32 字节及跨多块非对齐输入
- `mkHead()`：不得生成 `0xef` 开头或四个坏前缀
- Telegram Web：建立 session、收发消息、关闭 lane
- AES-CTR：固定 key/counter 下新旧实现输出一致

## 回滚

如需回滚，直接 revert commit：

```bash
git revert 365235b
```

或切回 `main`：

```bash
git checkout main
```