---
title: 'C++ Proxy Server Performance Optimization: From DNS Caching to TLS Session Reuse'
date: 2026-07-17T15:00:00+08:00
draft: false
categories: ["Tech"]
tags: ["C++", "Boost.Asio", "Networking", "Performance", "Proxy"]
summary: "Full-stack performance optimization for Overplus (a C++17 proxy server): global TCP DNS cache, SSL session reuse, buffer tuning, and log level demotion — 146 concurrent connections at just 25MB memory."
---

**Overplus** is a C++17 proxy server built on Boost.Asio, supporting SOCKS5, HTTPS, Trojan, and a custom V-Protocol. This post documents the full-stack performance optimization of its server component.

Source: https://github.com/xyanrch1024/overplus.git

---

## Baseline

| Metric | Value |
|--------|-------|
| Active connections | 27 |
| Memory usage | 5.6MB |
| DNS resolution | Every connection |
| TLS handshake | Full handshake every time |
| Disconnect logs | ERROR level, flooding |

Server: 2-core Intel Xeon Skylake, 2GB RAM, Ubuntu 24.04.

---

## Optimization 1: Global TCP DNS Cache

### The Problem

Every TCP connection triggers a DNS resolution. For frequently visited domains (e.g., `www.google.com`), this is pure waste.

### Wrong Approach: Per-Session Cache

The first attempt stored DNS cache as a `Session` member variable:

```cpp
// Session.h — WRONG
struct TcpDnsCacheEntry {
    tcp::endpoint endpoint;
    time_t expire_time;
};
std::unordered_map<std::string, TcpDnsCacheEntry> tcp_dns_cache_;
```

Looks reasonable, but `Session` is destroyed when the TCP connection ends — **the cache is destroyed with it**. Each Session only queries DNS once, writes the cache, then dies. Zero reuse.

### Correct Approach: Global DnsCacheManager Singleton

Extract the cache into a standalone singleton shared by all Sessions:

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
    std::mutex mtx_;
    std::unordered_map<std::string, TcpEntry> tcp_cache_;
};
```

Usage:

```cpp
// Session.cpp — do_resolve()
std::string dns_key = remote_host + ":" + remote_port;
tcp::endpoint cached_ep;
if (DnsCacheManager::instance().get_tcp(dns_key, cached_ep)) {
    do_connect(cached_ep);  // cache hit, skip DNS
    return;
}
// miss — resolve async, then cache
resolver_.async_resolve(remote_host, remote_port, ...);
```

Key design decisions:

- **`std::mutex`** protects concurrent access (multi-threaded io_context)
- **Lazy expiration**: TTL checked on lookup, expired entries replaced on next query
- **Active cleanup**: `steady_timer` scans every 600 seconds, removing expired entries
- **Configurable**: TTL and cleanup interval read from `server.json`, no hardcoding

### Cleanup Timer

```cpp
// Service.cpp
void Service::start_dns_cleanup_timer() {
    auto interval = ConfigManage::instance().server_cfg.dns_cleanup_interval;
    dns_cleanup_timer_.expires_after(std::chrono::seconds(interval));
    dns_cleanup_timer_.async_wait([this](const boost::system::error_code& ec) {
        if (ec) return;
        DnsCacheManager::instance().cleanup_expired();
        start_dns_cleanup_timer();
    });
}
```

### Configuration

```json
{
    "dns_cache_ttl": 600,
    "dns_cleanup_interval": 600
}
```

Defaults to 600 seconds (10 minutes) if not configured. Backward compatible.

---

## Optimization 2: SSL Session Reuse

### The Problem

Every client connection requires a full TLS handshake: certificate exchange + key negotiation, taking 2-5ms. For clients that reconnect frequently (browsers, mobile apps), this is avoidable overhead.

### Solution

Enable server-side session caching in the SSL context:

```cpp
// Service.cpp
ctx.set_options(
    boost::asio::ssl::context::default_workarounds
    | boost::asio::ssl::context::no_sslv2
    | boost::asio::ssl::context::single_dh_use);
SSL_CTX_set_session_cache_mode(ctx.native_handle(), SSL_SESS_CACHE_SERVER);
```

### How It Works

1. Client connects for the first time → full TLS handshake → server caches session (session ID + key parameters)
2. Client reconnects → sends session ID in ClientHello
3. Server recognizes the ID → skips full handshake, restores encrypted channel (<1ms)
4. OpenSSL defaults to caching 20,480 sessions — more than enough for our concurrency level

**No client-side changes required** — TLS session reuse is a protocol standard. Browsers, curl, and overplus_client all support it by default.

---

## Optimization 3: Buffer Size Tuning

### The Problem

Each Session allocates `in_buf` and `out_buf` for read/write. At 32KB, high-throughput scenarios trigger more frequent system calls.

### Solution

```cpp
// Session.h
static constexpr size_t MAX_BUFF_SIZE = 64 * 1024;  // 32KB → 64KB
```

Larger buffers mean each `async_read_some` / `async_write` processes more data. TLS records max at 16KB, so a 64KB buffer can hold multiple TLS records per read, reducing encryption/decryption context switches.

Memory cost: +64KB per Session. 146 connections ≈ 18MB — acceptable for a 2GB server.

---

## Optimization 4: Hot-Path Log Demotion

### The Problem

Disconnect error logs (`Connection reset by peer`, `stream truncated`) are **normal, expected behavior** in production. They should not be ERROR level. Under high concurrency, these logs flood the output — and each write involves `std::ostringstream` allocation, string formatting, and file I/O.

### Solution

Demote disconnect-related logs from `ERROR_LOG` to `DEBUG_LOG`:

| File | Line | Change |
|------|------|--------|
| `Server/Session.cpp:349` | `read from client` | ERROR → DEBUG |
| `Server/Session.cpp:367` | `read from downstream` | ERROR → DEBUG |
| `Server/Session.cpp:387` | `write to downstream` | ERROR → DEBUG |
| `Server/TlsSession.cpp:37` | `write to client (TCP)` | ERROR → DEBUG |
| `Server/TlsSession.cpp:55` | `write to client (UDP)` | NOTICE → DEBUG |

`DEBUG_LOG` macro evaluates to nothing when log level is above DEBUG — zero overhead:

```cpp
#define DEBUG_LOG \
    if (logger::get_log_level() <= L_DEBUG) \
    logger(__FILE__, __func__, __LINE__, L_DEBUG).stream()
```

---

## Results

| Metric | Before | After | Change |
|--------|--------|-------|--------|
| Active connections | 27 | 146 | **+440%** |
| Memory usage | 5.6MB | 24.9MB | +19.3MB (146×128KB buffers) |
| CPU | Normal | Near zero | — |
| DNS resolution | Every time | Cached, direct connect | **DNS round-trip eliminated** |
| TLS handshake | Full every time | Session restore on reconnect | **2-5ms saved** |
| ERROR logs | Flooding with disconnects | Zero | **Silent logs** |

25MB memory running 146 concurrent connections, CPU near zero.

---

## Takeaways

1. **Eliminate redundant work** — DNS cache avoids repeated resolution, SSL session reuse avoids repeated handshakes
2. **Global vs local cache** — per-session caches are useless in connection-per-session models; must be globally shared
3. **Thread safety** — shared caches under multi-threaded io_context need `std::mutex`
4. **Configuration over hardcoding** — TTL and cleanup intervals from JSON, tunable without recompilation
5. **Log discipline** — disconnects are normal behavior; ERROR-level logs for them pollute output and hurt performance

The biggest lesson: the first DNS cache implementation was wrong — stored per-Session, destroyed with it. **Understand object lifetimes before choosing cache placement.** That single insight drove the architecture from a useless local map to a proper global singleton.
