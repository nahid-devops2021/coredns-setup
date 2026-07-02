# CoreDNS Setup Documentation

## 1. Executive Summary

This document provides a comprehensive overview of the CoreDNS setup configured in the `coredns-setup` repository. CoreDNS is a cloud-native DNS server written in Go, designed to be flexible, extensible, and production-ready. This configuration demonstrates a containerized DNS infrastructure suitable for local lab environments with custom domain resolution, security filtering, and comprehensive monitoring capabilities.

---

## 2. What is CoreDNS?

CoreDNS is a modern DNS server that replaces traditional DNS solutions like BIND and Dnsmasq. Key characteristics include:

- **Cloud-Native Design**: Built for Kubernetes and microservice architectures
- **Plugin-Based Architecture**: Extensible through plugins that add functionality
- **Written in Go**: High performance, minimal resource overhead
- **Simple Configuration**: Uses a straightforward Corefile format instead of complex zone files
- **Observable**: Built-in metrics and logging for monitoring

### Use Cases
- Internal DNS for lab/dev environments
- Kubernetes DNS replacement
- DNS security gateway (blocking ads/malware)
- Internal service discovery
- DNS-based application routing

---

## 3. Repository Structure

```
coredns-setup/
├── Corefile                    # CoreDNS configuration file
├── Corefile_bk_02-july-2026   # Configuration backup
├── docker-compose.yaml         # Docker containerization config
├── db.nhlab.local             # Forward DNS zone for nhlab.local
├── db.reverse.zone            # Reverse DNS zone (192.168.0.x)
├── blocked.hosts              # Hosts list for ad/malware blocking
└── README.md                  # Documentation
```

---

## 4. Architecture Overview

### 4.1 DNS Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      DNS Clients                        │
│            (Systems querying for DNS)                   │
└─────────────┬───────────────���───────────────────────────┘
              │ DNS Queries (Port 53)
              ▼
┌─────────────────────────────────────────────────────────┐
│              CoreDNS Container (Port 53)                │
│  ┌─────────────────────────────────────────────────┐   │
│  │ Zone: nhlab.local (Internal Domain)             │   │
│  │  ├── dns.nhlab.local → 192.168.0.33             │   │
│  │  └── web.nhlab.local → 192.168.0.107            │   │
│  └─────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────┐   │
│  │ Reverse Zone: 192.168.0.x (PTR Records)         │   │
│  │  ├── 192.168.0.33 ← dns.nhlab.local             │   │
│  │  └── 192.168.0.107 ← web.nhlab.local            │   │
│  └─────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────┐   │
│  │ Security (Ad/Malware Blocking)                  │   │
│  │  └── blocked.hosts (filtered domains)           │   │
│  └─────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────┐   │
│  │ Caching Layer (300s TTL, 200k success entries)  │   │
│  └─────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────┐   │
│  │ Upstream Forwarding (Public Resolvers)          │   │
│  │  ├── 8.8.8.8 (Google DNS)                       │   │
│  │  ├─��� 8.8.4.4 (Google DNS Secondary)             │   │
│  │  └── 1.1.1.1 (Cloudflare DNS)                   │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
              │ Health Checks (Port 8053)
              │ Prometheus Metrics (Port 9153)
              ▼
     Monitoring & Observability
```

---

## 5. Core Configuration Explanation

### 5.1 Corefile Structure

The Corefile is CoreDNS's configuration file. It defines server blocks that handle DNS requests:

#### **Default Zone (.:53)**
```
.:53 {
    forward . 8.8.8.8 8.8.4.4 1.1.1.1 {
        health_check 5s
        max_concurrent 1000
    }
```

**Purpose**: Handles all DNS queries not matched by specific zones.

**Configuration Details**:
- **Forward Plugin**: Forwards unknown queries to public DNS servers
- **Health Check (5s)**: Validates upstream DNS health every 5 seconds
- **Max Concurrent (1000)**: Limits simultaneous upstream queries to prevent overload

#### **Security Layer (Ad/Malware Blocking)**
```
hosts /etc/coredns/blocked.hosts {
    fallthrough
}
```

**Purpose**: Blocks known ad servers and malware domains before forwarding.

**Configuration Details**:
- **Fallthrough**: If domain not in blocklist, continues to other plugins
- **Location**: `/etc/coredns/blocked.hosts` (empty in this setup, ready for population)

#### **Performance Caching**
```
cache 300 {
    success 200000
    denial 10000
    prefetch 20 1m 20% 
    serve_stale 1h
}
```

**Purpose**: Caches DNS responses to reduce upstream queries.

**Configuration Details**:
- **TTL (300s)**: Default cache time for valid responses
- **Success Capacity (200000)**: Maximum cached successful responses
- **Denial Capacity (10000)**: Maximum cached NXDOMAIN (not found) responses
- **Prefetch**: Refreshes popular domains before cache expiration
  - Query 20+ times in last minute → prefetch when 20% TTL remains
- **Serve Stale (1h)**: Returns cached data even if stale for up to 1 hour

#### **Monitoring & Logging**
```
log {
    class error
}
errors
reload
health :8053
prometheus :9153
```

**Purpose**: Observability and diagnostics.

**Configuration Details**:
- **Log**: Records errors only (not all queries)
- **Errors**: Logs error responses
- **Reload**: Allows configuration reload without restart
- **Health Check (port 8053)**: HTTP endpoint for liveness probes
- **Prometheus (port 9153)**: Exports metrics for monitoring

---

### 5.2 Internal DNS Zone (nhlab.local)

```
nhlab.local {
    file /etc/coredns/db.nhlab.local
    log {
        class error
    }
    errors
    reload
}
```

**Purpose**: Handles DNS queries for `nhlab.local` domain using local zone file.

**Zone File Content** (`db.nhlab.local`):
```dns
$ORIGIN nhlab.local.

@ IN SOA dns.nhlab.local. admin.nhlab.local. (
    2026061801 ; Serial
    7200       ; Refresh
    3600       ; Retry
    1209600    ; Expire
    3600       ; Minimum TTL
)

@    IN NS dns.nhlab.local.

dns  IN A 192.168.0.33
web  IN A 192.168.0.107
```

**Record Breakdown**:
| Type | Host | Value | Purpose |
|------|------|-------|---------|
| SOA | nhlab.local | admin.nhlab.local | Zone authority and contact |
| NS | nhlab.local | dns.nhlab.local | Nameserver for zone |
| A | dns.nhlab.local | 192.168.0.33 | DNS server itself |
| A | web.nhlab.local | 192.168.0.107 | Internal web server |

---

### 5.3 Reverse DNS Zone (PTR Records)

```
0.168.192.in-addr.arpa {
    file /etc/coredns/db.reverse.zone
    log {
        class error
    }
    errors
    reload
}
```

**Purpose**: Enables reverse DNS lookups (IP → hostname).

**Zone File Content** (`db.reverse.zone`):
```dns
$ORIGIN 0.168.192.in-addr.arpa.

@   IN SOA dns.nhlab.local. admin.nhlab.local. (
        2026062101
        7200
        3600
        1209600
        3600
)

@   IN NS dns.nhlab.local.

33  IN PTR dns.nhlab.local.
107 IN PTR web.nhlab.local.
```

**How Reverse DNS Works**:
- Query: `dig -x 192.168.0.33` 
- CoreDNS transforms to: `33.0.168.192.in-addr.arpa`
- Matches zone: `0.168.192.in-addr.arpa`
- Returns: `dns.nhlab.local`

---

## 6. Docker Deployment

### 6.1 Docker Compose Configuration

```yaml
services:  
  coredns:
    image: coredns/coredns:1.11.3
    container_name: coredns
    restart: unless-stopped
    command: -conf /etc/coredns/Corefile -dns.port 53
    ports:
      - "53:53"              # DNS over UDP
      - "53:53/udp"          # DNS over UDP (explicit)
      - "8053:8053"          # Health check endpoint
      - "9153:9153"          # Prometheus metrics
    volumes:
      - ./Corefile:/etc/coredns/Corefile
      - ./db.nhlab.local:/etc/coredns/db.nhlab.local
      - ./db.reverse.zone:/etc/coredns/db.reverse.zone
      - ./blocked.hosts:/etc/coredns/blocked.hosts
```

**Key Configuration Points**:
- **Image**: CoreDNS v1.11.3 (stable release)
- **Restart Policy**: `unless-stopped` (auto-restart on failure)
- **Port Mapping**:
  - **53 UDP**: Primary DNS protocol
  - **8053**: Health check HTTP endpoint
  - **9153**: Prometheus metrics scrape endpoint
- **Volume Mounts**: Maps local config files as read-only into container

---

## 7. DNS Query Flow

### 7.1 Forward DNS Query (name → IP)

```
User Query: nslookup web.nhlab.local

┌─ Query matches nhlab.local zone? ──────────────────────┐
│  YES: Use local zone file (db.nhlab.local)            │
│  └─ Return: web → 192.168.0.107                       │
└─────────────────────────────────────────────────────────┘

User Query: nslookup google.com

┌─ Query matches nhlab.local zone? ──────────────────────┐
│  NO: Continue to next handler                          │
│  ├─ Check cache (300s TTL) for google.com            │
│  │  ├─ HIT: Return cached result                      │
│  │  └─ MISS: Continue                                 │
│  ├─ Check blocked.hosts                               │
│  │  ├─ BLOCKED: Return NXDOMAIN                       │
│  │  └─ ALLOWED: Continue                              │
│  └─ Forward to upstream (8.8.8.8, 8.8.4.4, 1.1.1.1)  │
│     ├─ Wait for response                              │
│     ├─ Cache result (300s)                            │
│     └─ Return to client                               │
└─────────────────────────────────────────────────────────┘
```

### 7.2 Reverse DNS Query (IP → name)

```
User Query: nslookup -x 192.168.0.33

┌─ Transform to PTR format ──────────────────────────────┐
│  192.168.0.33 → 33.0.168.192.in-addr.arpa            │
└─────────────────────────────────────────────────────────┘

┌─ Query matches 0.168.192.in-addr.arpa zone? ──────────┐
│  YES: Use reverse zone file (db.reverse.zone)        │
│  └─ Return: 33 → dns.nhlab.local                      │
└─────────────────────────────────────────────────────────┘
```

---

## 8. Performance Tuning Explained

### 8.1 Cache Prefetch Strategy

The prefetch configuration (`prefetch 20 1m 20%`) implements intelligent cache warming:

**Algorithm**:
- Monitors query patterns over rolling 1-minute window
- If a domain receives 20+ queries → marked for prefetch
- When cached TTL drops to 20% remaining → automatically refresh
- Prevents cache misses for frequently accessed domains

**Benefits**:
- Reduces response time for popular queries
- Spreads upstream load evenly
- Improves user experience during traffic spikes

### 8.2 Stale Cache Serving

`serve_stale 1h` allows serving expired cached responses:

**When Used**:
- Upstream server is temporarily unavailable
- Network connectivity issues
- Upstream timeouts

**Example Flow**:
1. DNS record cached 2 hours ago (normally expired)
2. Upstream becomes unreachable
3. CoreDNS serves cached data (marked as stale)
4. Service continues operating despite DNS server failure
5. Window: Up to 1 hour beyond original expiration

---

## 9. Security Considerations

### 9.1 Ad/Malware Blocking

The `blocked.hosts` file enables DNS-level filtering:

**File Format** (standard hosts file):
```hosts
127.0.0.1 ads.tracking-domain.com
127.0.0.1 malware.evil-site.net
0.0.0.0 another-blocklist.org
```

**Implementation**:
- Queries for blocked domains → NXDOMAIN response
- Prevents client devices from connecting
- Works transparently across all applications

**Popular Blocklists**:
- AdGuard (50k+ ad domains)
- Pi-hole Lists (various categories)
- Malwarebytes Lists (malware/phishing)

### 9.2 Upstream Security

**Current Setup**:
```
forward . 8.8.8.8 8.8.4.4 1.1.1.1
```

**Considerations**:
- **Google DNS (8.8.8.8)**: Fast, logs queries for analytics
- **Cloudflare (1.1.1.1)**: Privacy-focused, DNSSEC enabled
- **No DNSSEC validation**: Upstream servers assumed trusted
- **Recommendation**: Consider adding `dnssec` plugin for DNSSEC validation

### 9.3 Access Control

**Current Setup**: Listens on all interfaces (0.0.0.0:53)

**Recommendations for Production**:
```corefile
# Restrict to specific interfaces
192.168.0.33:53 {
    # config
}

# Implement ACLs
acl {
    # Allow only internal IPs
    allow 192.168.0.0/24
    allow 10.0.0.0/8
}
```

---

## 10. Operational Tasks

### 10.1 Starting CoreDNS

```bash
# Start the container
docker-compose up -d

# Verify it's running
docker ps | grep coredns

# Check logs
docker logs coredns -f
```

### 10.2 Testing DNS Resolution

```bash
# Forward DNS test
nslookup web.nhlab.local localhost
dig web.nhlab.local @localhost

# Reverse DNS test
nslookup -x 192.168.0.107 localhost
dig -x 192.168.0.107 @localhost

# External domain test (forwarding)
nslookup google.com localhost
```

### 10.3 Modifying Zone Records

**Adding a new DNS record**:

1. Edit `db.nhlab.local`:
```dns
db  IN A 192.168.0.50
```

2. Increment the Serial number:
```dns
2026061802  ; Serial (increased from 2026061801)
```

3. Reload configuration:
```bash
# CoreDNS reloads on file change due to 'reload' plugin
# Or manually trigger: docker exec coredns ... (if enabled)
```

### 10.4 Monitoring Health

```bash
# Health check endpoint (HTTP)
curl http://localhost:8053/health

# Expected response: OK (200 status)

# Prometheus metrics
curl http://localhost:9153/metrics | grep coredns

# Key metrics to monitor:
# - coredns_dns_requests_total (query volume)
# - coredns_dns_responses_total (response breakdown)
# - coredns_cache_hits_total (cache effectiveness)
# - coredns_cache_misses_total (upstream load)
```

---

## 11. Advanced Topics

### 11.1 Scaling for Production

**Considerations**:
- **High Availability**: Deploy multiple CoreDNS instances with load balancing
- **Cache Optimization**: Increase cache sizes for high-traffic deployments
- **Upstream Diversity**: Use multiple DNS upstreams for resilience
- **Logging**: Shift from error-only to structured logging (JSON format)

### 11.2 Plugin Ecosystem

CoreDNS can be extended with plugins for:
- **Auto**: Kubernetes service discovery
- **Etcd**: Distributed DNS data backend
- **Route53**: AWS DNS integration
- **DNSSEC**: Security extensions
- **TLS/DoT**: Encrypted DNS transport

### 11.3 Zone Transfer (AXFR) Setup

For multi-server deployments:
```corefile
nhlab.local {
    file /etc/coredns/db.nhlab.local
    transfer out 192.168.0.50 {
        # Transfer zone to secondary DNS
    }
}
```

---

## 12. Troubleshooting Guide

| Issue | Diagnosis | Solution |
|-------|-----------|----------|
| DNS not resolving | `nslookup web.nhlab.local` returns "SERVFAIL" | Check Corefile syntax: `docker logs coredns` |
| Slow responses | Check cache hit ratio (Prometheus) | Increase cache size or prefetch threshold |
| Port already in use | `docker logs coredns` shows bind error | Change port mapping in docker-compose.yaml |
| Zone changes not taking effect | Query returns old data | Increment serial number in zone file |
| Health check failing | `curl http://localhost:8053/health` returns error | Check if CoreDNS process is running |

---

## 13. Conclusion

This CoreDNS setup provides a modern, containerized DNS infrastructure suitable for lab environments and internal service discovery. The configuration balances performance (caching, prefetch), security (ad-blocking, multiple upstreams), and observability (health checks, Prometheus metrics).

**Key Strengths**:
✅ Easy deployment with Docker Compose
✅ Intelligent caching with prefetch strategy
✅ Built-in monitoring capabilities
✅ Simple zone file management
✅ Reliable failover with serve_stale

**Next Steps**:
1. Populate `blocked.hosts` with ad/malware blocklists
2. Configure additional internal zones as needed
3. Set up Prometheus scraping for metrics
4. Implement alerting for health check failures
5. Document internal DNS naming conventions

---

**Document Version**: 1.0  
**Last Updated**: July 2, 2026  
**Repository**: https://github.com/nahid-devops2021/coredns-setup
