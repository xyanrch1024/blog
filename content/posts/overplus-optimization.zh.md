---
title: 'C++ 代理服务器性能优化实战：从 DNS 缓存到 TLS 复用的全链路调优'
date: 2026-07-17T15:00:00+08:00
draft: false
categories: ["技术"]
tags: ["C++", "Boost.Asio", "网络编程", "性能优化", "代理"]
summary: "对 Overplus（C++17 代理服务器）进行全链路性能优化：TCP DNS 全局缓存、SSL Session 复用、缓冲区调优、热路径日志降级，146 并发连接下内存仅 25MB。"
---

**Overplus** 是一个基于 C++17 和 Boost.Asio 的代理服务器，支持 SOCKS5、HTTPS、Trojan 和自定义 V-Protocol 协议。本文记录了对服务端进行的全链路性能优化过程。

源代码：https://github.com/xyanrch1024/overplus.git

---

## 优化前状态

| 指标 | 值 |
|------|-----|
| 活跃连接 | 27 |
| 内存占用 | 5.6MB |
| DNS 解析 | 每次连接重新解析 |
| TLS 握手 | 每次完整握手 |
| 断连日志 | ERROR 级别，大量刷屏 |

服务器配置：2 核 Intel Xeon Skylake，2GB RAM，Ubuntu 24.04。

---

## 优化一：TCP DNS 全局缓存

### 问题

每个 TCP 连接（Session）建立时都要做一次 DNS 解析。对于频繁访问的网站（如 `www.google.com`），这是完全不必要的重复工作。

### 错误尝试：per-session 缓存

最初的实现把 DNS 缓存放在 `Session` 类的成员变量里：

```cpp
// Session.h — 错误示范
struct TcpDnsCacheEntry {
    tcp::endpoint endpoint;
    time_t expire_time;
};
std::unordered_map<std::string, TcpDnsCacheEntry> tcp_dns_cache_;
```

看起来没问题，但 `Session` 在 TCP 连接结束时就销毁了——**缓存随 Session 一起销毁，根本没有跨连接复用的机会**。每个 Session 只查一次 DNS，缓存刚写入就没了。

### 正确方案：全局 DnsCacheManager 单例

把缓存提取为独立的全局单例，所有 Session 共享：

```cpp
// Shared/DnsCache.h
class DnsCacheManager : private boost::noncopyable {
public:
    struct TcpEntry {
        boost::asio::ip::tcp::endpoint endpoint;
        time_t expire_time;
    };
    static DnsCacheManager& instance();

    void set_default_ttl(time_t ttl);
    bool get_tcp(const std::string& key, boost::asio::ip::tcp::endpoint& ep);
    void put_tcp(const std::string& key, const boost::asio::ip::tcp::endpoint& ep);
    void cleanup_expired();

private:
    DnsCacheManager() = default;
    time_t default_ttl_ = 600;
    std::mutex mtx_;  // 线程安全
    std::unordered_map<std::string, TcpEntry> tcp_cache_;
};
```

使用时：

```cpp
// Session.cpp — do_resolve()
std::string dns_key = remote_host + ":" + remote_port;
tcp::endpoint cached_ep;
if (DnsCacheManager::instance().get_tcp(dns_key, cached_ep)) {
    do_connect(cached_ep);  // 命中缓存，跳过 DNS 解析
    return;
}
// 未命中，异步解析后写入缓存
resolver_.async_resolve(remote_host, remote_port, ...);
```

关键设计：

- **`std::mutex`** 保护并发访问（多线程 io_context）
- **惰性过期**：查询时检查 TTL，过期则重新解析并覆盖
- **主动清理**：`steady_timer` 每 600 秒扫描一次，删除过期条目
- **TTL 可配置**：从 `server.json` 读取，不硬编码

### 定时清理器

```cpp
// Service.cpp
void Service::start_dns_cleanup_timer() {
    auto interval = ConfigManage::instance().server_cfg.dns_cleanup_interval;
    dns_cleanup_timer_.expires_after(std::chrono::seconds(interval));
    dns_cleanup_timer_.async_wait([this](const boost::system::error_code& ec) {
        if (ec) return;
        DnsCacheManager::instance().cleanup_expired();
        start_dns_cleanup_timer();  // 递归定时
    });
}
```

### 配置

```json
{
    "dns_cache_ttl": 600,
    "dns_cleanup_interval": 600
}
```

不配置时默认 600 秒（10 分钟），向后兼容。

---

## 优化二：SSL Session 复用

### 问题

每次客户端连接都要做完整的 TLS 握手：证书交换 + 密钥协商，约 2-5ms。对于频繁重连的客户端（如浏览器），这是可避免的开销。

### 方案

在 SSL context 初始化时启用服务端 session 缓存：

```cpp
// Service.cpp
ctx.set_options(
    boost::asio::ssl::context::default_workarounds
    | boost::asio::ssl::context::no_sslv2
    | boost::asio::ssl::context::single_dh_use);
SSL_CTX_set_session_cache_mode(ctx.native_handle(), SSL_SESS_CACHE_SERVER);
```

### 原理

1. 客户端首次 TLS 握手，服务端完成完整握手并缓存 session（session ID + 密钥参数）
2. 客户端重连时在 ClientHello 里携带上次的 session ID
3. 服务端识别后跳过完整握手，直接恢复加密通道（<1ms）
4. OpenSSL 默认缓存 20480 个 session，对我们的并发量完全够用

**客户端无需任何改动**——TLS session 复用是协议标准行为，浏览器、curl、overplus_client 默认都支持。

---

## 优化三：缓冲区调整

### 问题

每个 Session 分配 `in_buf` 和 `out_buf` 两个缓冲区用于读写。原始大小 32KB，在高吞吐场景下会导致更频繁的系统调用。

### 方案

```cpp
// Session.h
static constexpr size_t MAX_BUFF_SIZE = 64 * 1024;  // 32KB → 64KB
```

更大的缓冲区意味着每次 `async_read_some` / `async_write` 能处理更多数据，减少系统调用次数。TLS record 最大 16KB，64KB 缓冲区可以容纳多个 TLS record，减少加密解密的上下文切换。

内存代价：每个 Session 多用 64KB。146 个连接 ≈ 18MB，对 2GB 服务器完全可以接受。

---

## 优化四：热路径日志降级

### 问题

客户端断开连接时触发的错误日志（`Connection reset by peer`、`stream truncated`）在生产环境中是**正常的、预期的行为**，不应该用 ERROR 级别。高并发时这些日志大量刷屏，每次写入都要分配 `std::ostringstream`、格式化字符串、写文件——严重影响性能。

### 方案

将断连相关日志从 `ERROR_LOG` 降级为 `DEBUG_LOG`：

| 文件 | 行 | 改动 |
|------|-----|------|
| `Server/Session.cpp:349` | `read from client` | ERROR → DEBUG |
| `Server/Session.cpp:367` | `read from downstream` | ERROR → DEBUG |
| `Server/Session.cpp:387` | `write to downstream` | ERROR → DEBUG |
| `Server/TlsSession.cpp:37` | `write to client (TCP)` | ERROR → DEBUG |
| `Server/TlsSession.cpp:55` | `write to client (UDP)` | NOTICE → DEBUG |

`DEBUG_LOG` 宏在日志级别高于 DEBUG 时完全不执行，零开销：

```cpp
#define DEBUG_LOG \
    if (logger::get_log_level() <= L_DEBUG) \
    logger(__FILE__, __func__, __LINE__, L_DEBUG).stream()
```

---

## 结果对比

| 指标 | 优化前 | 优化后 | 变化 |
|------|--------|--------|------|
| 活跃连接 | 27 | 146 | **+440%** |
| 内存占用 | 5.6MB | 24.9MB | +19.3MB（146×128KB buffer） |
| CPU | 正常 | 几乎为零 | — |
| DNS 解析 | 每次重做 | 缓存命中直接连接 | **省掉 DNS 往返** |
| TLS 握手 | 每次完整握手 | 重连时 session 恢复 | **省掉 2-5ms** |
| ERROR 日志 | 大量断连刷屏 | 零 ERROR | **日志安静** |

25MB 内存跑 146 个并发连接，CPU 几乎为零。

---

## 总结

这次优化的核心思路：

1. **消除重复工作** — DNS 缓存避免重复解析，SSL Session 复用避免重复握手
2. **全局共享 vs 局部缓存** — per-session 缓存在连接模型下是无效的，必须全局共享
3. **线程安全** — 多线程 io_context 下共享缓存需要 `std::mutex` 保护
4. **配置化** — TTL、清理间隔从 JSON 读取，不硬编码，方便调优
5. **日志分级** — 断连是正常行为，不应该用 ERROR 级别污染日志

性能优化不是一蹴而就的——第一次把 DNS 缓存放在 Session 成员里是错的，后来才意识到 Session 销毁时缓存也跟着没了。**先理解生命周期，再选择缓存位置**，这是这次最大的教训。
