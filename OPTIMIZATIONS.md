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

## 第三轮

8. KWS 拨号错误上下文

> 第三轮完整实现见 commit `b70288e`。

## 1. leases / macKeys GC

（内容同前，略）

## 8. KWS 拨号错误上下文

### 问题

原 `dial()` 在两个 KWS host 都失败时只抛出裸 `Error()`，不保留任何原因、host、DC 或 media 信息，排障困难。

### 修改

```js
async function dial({ target, media, signal }) {
  let lastErr = null;
  for (const host of hosts(target + 1, media)) {
    if (signal?.aborted) throw Error();
    const url = `wss://${host}/apiws`;
    let kws;
    try {
      kws = mkWire(new WebSocket(url, 'binary'), 7500, signal);
      await kws.ready;
      if (signal?.aborted) throw Error();
      return kws;
    } catch (err) {
      lastErr = err;
      kws?.close();
      if (signal?.aborted) throw err;
    }
  }
  throw new Error(
    `kws dial failed dc=${target + 1} media=${media}`,
    { cause: lastErr },
  );
}
```

## 回归测试

- KWS 双 host 均失败时，错误信息包含 dc 与 media
- 单 host 失败、另一 host 成功时正常建立连接
- Telegram Web：建立 session、收发消息、关闭 lane

## 回滚

回滚第三轮：

```bash
git revert b70288e
```