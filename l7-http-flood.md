# Incident Report: L7 HTTP Flood — Caught by WAF, Not DDoS Scrubbing

> **Severity:** P1 | **Duration:** 2 hrs 14 mins | **Status:** Resolved

---

## What Happened

An e-commerce customer reported slow page loads. The DDoS platform showed nothing unusual — traffic and bandwidth were normal. Checked WAF logs and found an HTTP GET flood hitting `/product` and `/checkout` at 18,000 requests/sec from 340 IPs.

The attack was designed to stay under volumetric thresholds. It wasn't trying to fill the pipe — it was exhausting the web server's CPU and database connections at the application layer.

---

## Timeline

| Time  | Event |
|-------|-------|
| 11:03 | Customer reports slow load times |
| 11:08 | DDoS platform checked — no alerts, traffic normal |
| 11:11 | WAF logs checked — spike in block events found |
| 11:14 | HTTP GET flood confirmed — 18,000 req/sec to `/product` and `/checkout` |
| 11:17 | 340 source IPs identified, 50 IPs responsible for 68% of traffic |
| 11:22 | WAF rate limit applied — 100 req/min per IP on affected URIs |
| 11:28 | Customer confirms site performance recovering |
| 11:35 | Top 50 attacking IPs hard-blocked in WAF |
| 12:44 | Attack traffic drops to baseline |
| 13:17 | Rate limit rule removed after 30-min cool-down |

---

## What the WAF Logs Showed

```
Window:           11:01 – 11:14 (5 minutes)
Total Requests:   54,000
Blocked:          41,200  (76%)
Allowed:          12,800  (24%)

Top URIs:
  /product     →  31,400 requests
  /checkout    →  19,600 requests

Source IPs:       340 unique IPs
Avg rate/IP:      52 req/sec  (legit users: 1–3 req/sec)
HTTP Method:      GET only
```

---

## Why DDoS Scrubbing Didn't Catch It

The DDoS platform monitors bandwidth and packet rates at the network layer (L3/L4).  
This attack kept total bandwidth normal — it was only hitting hard at the application layer (L7).

**DDoS scrubbing and WAF solve different problems:**

| Tool | What it catches |
|------|----------------|
| DDoS Scrubbing | Volumetric floods — UDP, SYN, ICMP, amplification |
| WAF | Application floods — HTTP floods, bot traffic, request abuse |

---

## How It Was Fixed

**Step 1 — Rate limit on targeted URIs**
```
Rule:       Rate Limit
URIs:       /product, /checkout
Threshold:  100 requests/min per source IP
Action:     Block — return 429 Too Many Requests
```

Legitimate users send 1–5 req/min. Setting 100/min blocks bots while keeping real users unaffected.

**Step 2 — Hard block top attacking IPs**
```
IPs blocked:   50
Block type:    Hard block — all traffic dropped
Review after:  24 hours
```

---

## Root Cause

Targeted L7 HTTP GET flood from a small botnet (~340 IPs).  
Attacker deliberately stayed under volumetric thresholds to avoid DDoS detection.  
Goal was to exhaust web server resources — not fill the network pipe.

---

## What Could Have Been Done Better

- Check WAF logs first for application-layer complaints — not DDoS platform
- `/checkout` and `/login` should always have baseline rate limiting pre-configured
- Customer had no bot protection or CAPTCHA on checkout — flagged as recommendation

---

## Key Takeaway

> If the DDoS platform shows nothing but the site is down — go straight to WAF logs.  
> L7 floods are built to stay invisible at the network layer.

---

*Author: NOC Engineer (L2) | 2024 | Details anonymized*
