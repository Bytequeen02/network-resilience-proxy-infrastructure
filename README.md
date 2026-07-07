# OSINT Network Resilience Client (Pillar 3)

Welcome to the Network Infrastructure module for the Enterprise OSINT & Data Crawling Pipeline. 
This package provides a highly automated, anti-bot resistant, and horizontally scalable HTTP client.

## 🏗 Architecture Overview

The `NetworkClient` acts as a facade, abstracting away the extreme complexity of network evasion. 
When a downstream scraper calls `client.get()`, the request passes through multiple resilient layers:

1. **Concurrency Limiter (Semaphore):** Prevents OS socket exhaustion.
2. **Domain Rate Manager (Token Bucket):** Ensures ethical scraping and evades rate-limit bans.
3. **Entropy Generator:** Injects human-like micro-delays between requests.
4. **Session Manager:** Isolates cookies and maintains connection pools.
5. **Proxy Manager:** Assigns a thread-safe rotating or sticky IP address.
6. **Camouflage Engine (Headers/UA):** Generates strict, mathematically consistent Chrome/TLS fingerprints.
7. **Execution & Telemetry:** Dispatches the request and logs performance securely.
8. **WAF Detector:** Inspects payloads for Datadome, Akamai, or Cloudflare JS challenges.
9. **Retry Engine (Tenacity):** Uses Exponential Backoff + Jitter to silently recover from network failures.

## 🚀 Quick Start (Usage Guide)

### 1. Installation
Install the required dependencies via Pip:
```bash
pip install -r requirements.txt
```

### 2. Configuration (`.env`)
Create a `.env` file in the root directory:
```env
MAX_RETRIES=5
CONNECT_TIMEOUT=5.0
READ_TIMEOUT=15.0
LOG_LEVEL=INFO
# Supply proxies directly or via a file
PROXIES=http://user:pass@proxy1.com:8000,http://user:pass@proxy2.com:8000
```

### 3. Basic Synchronous Usage
For stateful, multi-step scraping tasks (e.g., Logging into a website):
```python
from network import NetworkClient

client = NetworkClient()

# By providing a session_id, the client guarantees:
# 1. The exact same Proxy IP will be used for both requests.
# 2. Cookies from the login will carry over to the dashboard.
login_resp = client.post("https://target.com/login", session_id="user_123", json={"user": "admin"})
data_resp = client.get("https://target.com/dashboard", session_id="user_123")

print(data_resp.text)
```

### 4. High-Performance Asynchronous Usage
For massive, parallel data extraction (e.g., fetching 10,000 product URLs):
```python
import asyncio
from network import AsyncNetworkClient

async def fetch_all(urls):
    client = AsyncNetworkClient()
    
    # The client's internal Semaphore safely limits concurrency
    tasks = [client.get(url) for url in urls]
    responses = await asyncio.gather(*tasks)
    
    # Always clean up async connections!
    await client.close_all()
    return responses
```

## 👨‍💻 Developer Guide

If you are modifying the internal network logic, adhere to the following rules:
1. **Never use raw `requests.get()`:** It does not utilize connection pooling. Use the `SessionManager`.
2. **Thread Safety:** All modifications to global state (proxies, rate limits) MUST use `threading.Lock()`.
3. **No Credential Logging:** Ensure `logger.py` masking is active before printing proxy URLs.
4. **Testing:** Run `pytest tests/` before opening a Pull Request.

## 🚢 Deployment Guide

This package is designed to run in a containerized environment (Docker/Kubernetes).
1. Ensure the `logs/` directory is mounted to an external volume, otherwise logs will be destroyed on pod restart.
2. Adjust `MAX_CONNECTIONS` in the `AsyncNetworkClient` based on the CPU/RAM limits of your specific Kubernetes Pod.

## 🛡️ Core Evasion Mechanisms

### TLS Fingerprinting
Standard Python HTTP libraries (`requests`, `httpx`, `aiohttp`) use default OpenSSL fingerprints that are instantly flagged by modern anti-bot systems (Cloudflare, Datadome, Akamai). To bypass this, we use `curl_cffi`, which intercepts the TLS handshake and perfectly impersonates the cipher suites, extensions, and ALPN negotiation of modern browsers (e.g., Chrome 124).

### SSL Verification
We strictly control SSL verification to prevent unnecessary failures or security vulnerabilities. It is managed via the `VERIFY_SSL` configuration in `config.py` (default: `True`). We do not use hardcoded `verify=False` anywhere in the codebase.

### Proxy Lifecycle & Rotation
Proxies are central to maintaining network resilience:
- **Sticky Sessions:** Requests sharing a `session_id` are pinned to a specific proxy to maintain state.
- **Auto-Rotation:** Upon network failure or WAF block, the currently active proxy is placed into a cooldown period (e.g., 60-300 seconds), and a fresh proxy is immediately rotated in for the subsequent retry.
- **Permanent Removal:** Proxies that repeatedly fail (configurable threshold, default 5 failures) are permanently purged from the rotation pool and from active sticky sessions.

### User-Agent Rotation
User-Agents dictate how the target server perceives our client.
- **Modern Browsers:** Our rotation pool only includes modern variants of Chrome, Firefox, Edge, and Safari across Windows, macOS, Linux, Android, and iOS.
- **Smart Rotation:** The `UserAgentManager` remembers the last generated User-Agent to ensure consecutive requests do not repeat the same identity whenever possible.

### Intelligent Retry Mechanism
The network engine uses `tenacity` for resilient execution.
- **Exponential Backoff with Jitter:** Prevents thundering herd problems by adding random delays between retries.
- **Fresh Contexts:** If a request fails, the proxy is automatically flagged as unhealthy, and the Retry Engine ensures that the next attempt uses a completely new proxy IP while seamlessly maintaining the session cookies.
