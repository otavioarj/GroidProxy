# Groid – Golang Android Proxier

## 📦 Usage

```bash
./groidproxy [options] [packages...]
```

## ⚙️ Options

| Flag               | Description                                                                                                      |
|--------------------|------------------------------------------------------------------------------------------------------------------|
| `-blacklist string`| Comma-separated list of blocked hosts/IPs (`.domain.com` for wildcards)<br>❗Doesn't work on raw redirect        |
| `-d`               | Run as daemon                                                                                                    |
| `-dns`             | Also redirect DNS (port 53)                                                                                      |
| `-flush`           | Remove all GROID rules                                                                                           |
| `-global`          | Redirect all traffic                                                                                             |
| `-list`            | List current rules                                                                                               |
| `-local-port int`  | Local port for transparent proxy (default: `8123`)                                                               |
| `-p string`        | Proxy address (`host:port`, `http://host:port`, or `socks5://host:port`)                                         |
| `-remove string`   | Remove rules for specified package                                                                               |
| `-save string`     | Save traffic to a SQLite database<br>❗Doesn't work on raw redirect<br>⭐Can work without external proxy (default: `/data/local/tmp/Groid.db`) |
| `-stats`           | Show I/O statistics                                                                                              |
| `-timeout int`     | Connection timeout in seconds (default: `10`)                                                                    |
| `-tlscert string`  | PKCS12 certificate for TLS interception AND CA per-host                                                          |
| `-tlspass string`  | Password for PKCS12 certificate                                                                                  |
| `-v`               | Verbose output                                                                                                   |

## 🧰 Proxy Modes

- `host:port` — Redirect TCP packets (raw redirect) to an external transparent proxy  
- `http://host:port` — Transparent redirect to an HTTP proxy  
- `socks5://host:port` — Transparent redirect to a SOCKS5 proxy  
- `save /path/base.db` — Save all HTTP traffic (app ⇄ server) to SQLite file  
  - Can work without upstream proxy  
  - With TLS (PKCS12) certificate, saves decrypted HTTPS content

## 🔍 Examples

```bash
./groidproxy -p 192.168.1.100:8888 com.example.app
./groidproxy -p http://192.168.1.100:8080 com.example.app com.android.chrome
./groidproxy -p socks5://192.168.1.100:1080 -global
./groidproxy -p socks5://192.168.1.100:1080 -blacklist "facebook.com,.youtube.com" com.example.app
./groidproxy -p http://192.168.1.100:1080 -save /data/local/tmp/Example.db -tlscert burp.pk12 -tlspass pass com.example.app
./groidproxy -save /data/local/tmp/Example.db -tlscert burp.pk12 -tlspass pass com.example.app
```
# 🐛 Known bugs

## Burp HTTP State machine

Until Groid v1.2, there're bugs related to how Burp sends an EOF (end-of-file) after each HTTP/1.1 response, this was **fixed by version 1.3.0 :)**.
If you encounter any issues while using Groid HTTP mode (-p http://), **OPEN** a ticket. In **v1.2** one of the following configs must be set to use HTTP mode (-p http://):
- Proxy Settings -> Proxy listeners -> Request handling -> Support invisible proxying: **ON** (Best)
- Proxy Settings -> Miscellaneous: Use keep-alvie for HTTP/1.1 **OFF** (Worst)
- Proxy Settings -> Miscellaneous: Set response header "Connection: close" **ON** 

# 🔨 Build

First run will sync mod dependencies and build for AARCH64: 
```
/build.sh    
 Building...  
 Done :)  
```

Build and push ELF to Android /data/local/tmp using ADB:
```
/build.sh push  
 Building...
 Done :)
 groidproxy: 1 file pushed, 0 skipped. 255.0 MB/s (8323224 bytes in 0.031s)   
 Pushed to /data/local/tmp  
```

If need to resync mod dependencies:
```
 /build.sh update
 [go mod tidy output :P]
```

# 🧱 Groid Proxy - Architecture

## Overview

Groid is a transparent proxy for Android that intercepts and redirects application traffic through iptables, with support for TLS capture and SQLite database storage.

## Operation Modes

### 1. Direct Redirect Mode (`-p host:port`)

```
┌─────────────┐    iptables    ┌──────────────────┐    TCP    ┌─────────────┐
│ Android App │ ──────────────►│ DNAT Redirection │ ────────► │ Ext. Proxy  │
└─────────────┘                └──────────────────┘           └─────────────┘
      │                                                              │
      │                        Raw TCP Passthrough                   │
      └──────────────────────────────────────────────────────────────┘
```

**Features:**
- ✅ **Direct redirection** via iptables DNAT
- ✅ **Zero overhead** - no local processing
- ✅ **Compatible** with any transparent proxy
- ❌ **No data capture** capabilities

---

### 2. HTTP Proxy Mode (`-p http://host:port`)

```
┌─────────────┐    iptables    ┌──────────────────┐    HTTP     ┌─────────────┐
│ Android App │ ──────────────►│ Local Proxy      │ ──────────► │ HTTP Proxy  │
└─────────────┘                │ (Port 8123)      │  CONNECT    └─────────────┘
      │                        └──────────────────┘                    │
      │                                │                               │
      │        Transparent TCP         │        HTTP Protocol          │
      └────────────────────────────────┼───────────────────────────────┘
                                       │
                                ┌──────▼──────┐
                                │ HTTP Parser │
                                │ & Relay     │
                                └─────────────┘
```

**Data Flow:**
1. **App** → TCP connection → **iptables** → redirect to `127.0.0.1:8123`
2. **Local Proxy** → reads original destination → creates HTTP CONNECT to external proxy
3. **HTTP Proxy** → establishes tunnel → **Target Server**
4. **Bidirectional relay** between app and server through proxy chain

---

### 3. SOCKS5 Proxy Mode (`-p socks5://host:port`)

```
┌─────────────┐    iptables    ┌──────────────────┐   SOCKS5    ┌─────────────┐
│ Android App │ ──────────────►│ Local Proxy      │ ──────────► │ SOCKS5 Proxy│
└─────────────┘                │ (Port 8123)      │ Handshake   └─────────────┘
      │                        └──────────────────┘                    │
      │                                │                               │
      │        Transparent TCP         │     SOCKS5 Protocol           │
      └────────────────────────────────┼───────────────────────────────┘
                                       │
                                ┌──────▼──────┐
                                │ SOCKS5      │
                                │ Handler     │
                                └─────────────┘
```

**Protocol Flow:**
1. **App** → TCP connection → **iptables** → redirect to `127.0.0.1:8123`
2. **Local Proxy** → SOCKS5 handshake (version + auth) → **SOCKS5 Proxy**
3. **Connection request** → target host:port → **Target Server**
4. **Bidirectional relay** between app and server through SOCKS5 tunnel

---

### 4. Capture Mode (`-save database.db` + optional proxy)

#### 4.1 Direct Capture Mode (`-save database.db` only)

```
┌─────────────┐    iptables    ┌──────────────────┐   Direct   ┌─────────────┐
│ Android App │ ──────────────►│ TLS Interceptor  │ ─────────► │ Target      │
└─────────────┘                │ (Port 8123)      │    TLS     │ Server      │
      │                        └──────────────────┘            └─────────────┘
      │                                │
      │           TLS Tunnel 1         │         TLS Tunnel 2
      └────────────────────────────────┼─────────────────────────────────────
                                       │
                               ┌───────▼─────────┐
                               │ TLS Interceptor │
                               │ ┌─────────────┐ │
                               │ │ Certificate │ │
                               │ │ Generator   │ │
                               │ └─────────────┘ │
                               │ ┌─────────────┐ │
                               │ │ HTTPPairer  │ │
                               │ │ FIFO Queue  │ │
                               │ └─────────────┘ │
                               │ ┌─────────────┐ │
                               │ │ SQLite      │ │
                               │ │ Worker      │ │
                               │ └─────────────┘ │
                               └─────────────────┘
```

#### 4.2 Capture + Proxy Mode (`-save database.db -p http://host:port`)

```
┌─────────────┐    iptables    ┌──────────────────┐    HTTP     ┌─────────────┐   ┌─────────────┐
│ Android App │ ──────────────►│ TLS Interceptor  │ ──────────► │ HTTP Proxy  │──►│ Target      │
└─────────────┘                │ (Port 8123)      │  CONNECT    └─────────────┘   │ Server      │
      │                        └──────────────────┘                               └─────────────┘
      │                                │
      │           TLS Tunnel 1         │         TLS Tunnel 2
      └────────────────────────────────┼─────────────────────────────────────────────────────────
                                       │
                               ┌───────▼─────────┐
                               │ TLS Interceptor │
                               │ ┌─────────────┐ │
                               │ │ SNI Extract │ │
                               │ │ & Cert Gen  │ │
                               │ └─────────────┘ │
                               │ ┌─────────────┐ │
                               │ │ HTTPPairer  │ │
                               │ │ Req/Resp    │ │
                               │ │ Correlation │ │
                               │ └─────────────┘ │
                               │ ┌─────────────┐ │
                               │ │ SQLite DB   │ │
                               │ │ Async Save  │ │
                               │ └─────────────┘ │
                               └─────────────────┘
```

---

## TLS Interception Architecture

### Certificate Handling

```
┌─────────────────┐    ClientHello     ┌─────────────────┐
│   Android App   │ ─────────────────► │  TLS Proxy      │
└─────────────────┘                    └─────────────────┘
                                              │
                                       ┌──────▼──────┐
                                       │ Extract SNI │
                                       │gateway.com  │
                                       └──────┬──────┘
                                              │
                                    ┌─────────▼─────────┐
                                    │ Generate Cert     │
                                    │ CN: gateway.com   │
                                    │ Signed by Root CA │
                                    └─────────┬─────────┘
                                              │
┌─────────────────┐   ServerHello + Cert      │
│   Android App   │ ◄─────────────────────────┘
└─────────────────┘
```

### HTTP Request/Response Pairing

```
Client Goroutine:                    Server Goroutine:
┌─────────────────┐                  ┌─────────────────┐
│ Read from App   │                  │ Read from Srv   │
│      ↓          │                  │      ↓          │
│ Relay to Server │                  │ Relay to App    │
│      ↓          │                  │      ↓          │
│ Buffer Request  │                  │ Buffer Response │
│      ↓          │                  │      ↓          │
│ Complete HTTP?  │                  │ Complete HTTP?  │
│      ↓          │                  │      ↓          │
│ Add to Queue    │                  │ Match & Pair    │
└─────────────────┘                  └─────────────────┘
         │                                    │
         └─────────────┐      ┌───────────────┘
                       ▼      ▼
                ┌─────────────────┐
                │   HTTPPairer    │
                │ ┌─────────────┐ │
                │ │ Pending     │ │
                │ │ Requests    │ │
                │ │ FIFO Queue  │ │
                │ └─────────────┘ │
                │ ┌─────────────┐ │
                │ │ Save Worker │ │
                │ │ Channel     │ │
                │ └─────────────┘ │
                └─────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │ SQLite Database │
                │ ┌─────────────┐ │
                │ │  requests   │ │
                │ │ ┌─────────┐ │ │
                │ │ │timestamp│ │ │
                │ │ │ method  │ │ │
                │ │ │   url   │ │ │
                │ │ │ request │ │ │
                │ │ │response │ │ │
                │ │ └─────────┘ │ │
                │ └─────────────┘ │
                └─────────────────┘
```

---

## Performance Characteristics

### Relay Priority System

```
Priority 1: Data Relay (Never Blocks)
┌─────────────────────────────────────┐
│ client.Read() → server.Write()      │
│ server.Read() → client.Write()      │
└─────────────────────────────────────┘
                 ↓
Priority 2: Capture Processing
┌─────────────────────────────────────┐
│ Buffer accumulation                 │
│ HTTP message parsing                │
│ Request/Response pairing            │
└─────────────────────────────────────┘
                 ↓
Priority 3: Database Storage
┌─────────────────────────────────────┐
│ Async worker goroutines             │
│ Non-blocking channel operations     │
│ SQLite batch operations             │
└─────────────────────────────────────┘
```

### Resource Management

- **Memory**: Bounded buffers with automatic cleanup
- **Goroutines**: One per connection + async workers
- **Database**: Async writes with channel buffering
- **Timeouts**: 30-second orphan request cleanup
