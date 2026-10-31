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

> 第三轮完整实现见 commit `b70288e`，已通过 PR #3 合并到 `main`（merge commit `244b7c1`）。

## 第四轮

9. 下游大帧发送
10. KWS 写入背压
11. `mkPkt()` 0xee/0xdd 解析优化
12. `cipher()` 分配优化
13. `dial()` 重试退避

> 第四轮完整实现见 commit `dfee0b1`。

## 1. leases / macKeys GC

（内容同前，略）

## 8. KWS 拨号错误上下文

（内容同前，略）

## 9. 下游大帧发送

### 问题

原 `emitDn()` 固定按 64KB 切帧，帧头开销和 `send` 次数偏多。

### 修改

```js
const n = Math.min(1024 * 1024, b.byteLength - o, st.out);
```

## 10. KWS 写入背压

### 问题

原 `mkWire.write` 未检查 `ws.bufferedAmount`，高吞吐时可能内存堆积。

### 修改

```js
while (ws.bufferedAmount > CFG.wsBufMax) {
  await new Promise(ok => setTimeout(ok, 1));
  if (closed) throw fault || Error();
}
```

## 11. mkPkt() 0xee/0xdd 解析优化

### 问题

原实现每个包都新建 `DataView`。

### 修改

```js
pLen = (tail[o] | tail[o + 1] << 8 | tail[o + 2] << 16 | tail[o + 3] << 24) & 0x7fffffff;
```

## 12. cipher() 分配优化

### 问题

原实现无论是否 reverse 都会 `dup()` 复制 48 字节。

### 修改

```js
const src = u8(head).subarray(8, 56);
const mat = reverse ? (m => (m.reverse(), m))(dup(src)) : src;
```

## 13. dial() 重试退避

### 问题

原实现双 host 失败直接抛错，弱网下成功率低。

### 修改

```js
for (let attempt = 0; attempt < 2; attempt++) {
  for (const host of hosts(target + 1, media)) { ... }
  if (attempt === 0 && !signal?.aborted) await new Promise(ok => setTimeout(ok, 250));
}
```

## 回归测试

- 大流量下载时下游单帧最大 1MB
- KWS `bufferedAmount` 超阈值时写入等待
- 0xee/0xdd 模式解析结果与原实现一致
- 双 host 均失败时重试一次（250ms 退避）
- Telegram Web：建立 session、收发消息、关闭 lane

## 回滚

回滚第四轮：

```bash
git revert dfee0b1
```