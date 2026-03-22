# OpenClaw-web_fetch-SSRF-Guard-Fix-for-Shadowrocket-Clash-fake-ip-Environments
OpenClaw web_fetch SSRF Guard Fix for Shadowrocket/Clash fake-ip Environments
**Tested version:** OpenClaw 2026.3.13 (61d171a)
**Affected environment:** macOS + Shadowrocket (or Clash/Mihomo) in fake-ip DNS mode
**Created:** 2026-03-18

---

## 1. Problem Description

The `web_fetch` tool fails on all external URLs with the following error:

```json
{"status":"error","tool":"web_fetch","error":"Blocked: resolves to private/internal/special-use IP address"}
```

`web_search` works fine in the same environment. Browsers work fine.

---

## 2. Root Cause Analysis

### 2.1 Shadowrocket's fake-ip Mechanism

Shadowrocket intercepts all DNS queries at the TUN layer and returns virtual IP addresses in the 198.18.x.x range for all domains. When an application connects to 198.18.x.x, Shadowrocket intercepts the connection at the network layer, re-resolves DNS with the real resolver, and forwards the request through the proxy server. 198.18.0.0/15 is the RFC 2544 benchmark address range.

To verify:

```bash
nslookup www.reuters.com
```

If the result shows a 198.18.x.x address, Shadowrocket fake-ip is active.

### 2.2 OpenClaw's SSRF Guard Mechanism

Before making any network request, `web_fetch` calls `resolvePinnedHostnameWithPolicy`, which performs a DNS lookup and then checks whether the resolved IP belongs to a private/special-use address range. 198.18.x.x is classified as RFC 2544 special-use and triggers a `SsrFBlockedError`.

The key code inside `resolvePinnedHostnameWithPolicy`:

```javascript
const skipPrivateNetworkChecks = allowPrivateNetwork || isExplicitAllowed;
// ...DNS resolution...
if (!skipPrivateNetworkChecks) assertAllowedResolvedAddressesOrThrow(results, params.policy);
```

When `skipPrivateNetworkChecks` is `false`, `assertAllowedResolvedAddressesOrThrow` sees 198.18.x.x, throws an exception, and all subsequent code (including the actual network request) never executes.

### 2.3 Full Chain

```
Agent wants to fetch github.com
  → DNS resolves to 198.18.0.36 (Shadowrocket fake-ip)
  → resolvePinnedHostnameWithPolicy checks the IP
  → Classified as special-use address
  → Throws SsrFBlockedError
  → web_fetch returns blocked error
```

`web_search` works because it uses the trusted mode (`withTrustedWebToolsEndpoint`), which passes `WEB_TOOLS_TRUSTED_NETWORK_SSRF_POLICY` to skip this check. `web_fetch` uses the strict mode (`withStrictWebToolsEndpoint`), which does not pass this policy.

---

## 3. Fix Location

### 3.1 File Path

All files to modify are in:

```
/opt/homebrew/lib/node_modules/openclaw/dist/
```

### 3.2 Files to Modify

| File | Line | Note |
|------|------|------|
| `reply-Bm8VrLQh.js` | 16519 | Main web_fetch path |
| `auth-profiles-DDVivXkv.js` | 17579 | Duplicate |
| `auth-profiles-DRjqKE3G.js` | 15350 | Duplicate |
| `discord-CcCLMjHw.js` | 9897 | Duplicate |
| `model-selection-46xMp11W.js` | 18763 | Duplicate |
| `model-selection-CU2b7bN6.js` | 18296 | Duplicate |

> Note: Filenames contain hash values and will differ across versions. The fix is identical for all files.

---

## 4. Fix Method

### 4.1 Target Statement

In each file, locate the following line inside the `resolvePinnedHostnameWithPolicy` function:

```javascript
if (!skipPrivateNetworkChecks) assertAllowedResolvedAddressesOrThrow(results, params.policy);
```

### 4.2 Modification

Comment out the original line and add a new line below it:

```javascript
//if (!skipPrivateNetworkChecks) assertAllowedResolvedAddressesOrThrow(results, params.policy);
if (!skipPrivateNetworkChecks && !results.every(r => r.address?.startsWith("198.18."))) assertAllowedResolvedAddressesOrThrow(results, params.policy);
```

### 4.3 Logic Explanation

- **Original:** If `skipPrivateNetworkChecks` is `false`, run private network checks on all resolved IPs. 198.18.x.x gets blocked.
- **Modified:** If `skipPrivateNetworkChecks` is `false` AND the resolved IPs are not all 198.18.x.x, run private network checks. This exempts only Shadowrocket's fake-ip range (198.18.0.0/15). Other private addresses (127.x, 192.168.x, 10.x) are still blocked.

---

## 5. Step-by-Step Execution

1. Open the 6 files in VS Code, use `Cmd+G` to jump to the target line number
2. Comment out the original line by adding `//` at the beginning
3. Add the new conditional statement below the commented line
4. Save all files
5. Run `openclaw gateway restart`
6. Test `web_fetch` with any external URL

Commands to open files:

```bash
open -a "Visual Studio Code" /opt/homebrew/lib/node_modules/openclaw/dist/reply-Bm8VrLQh.js
open -a "Visual Studio Code" /opt/homebrew/lib/node_modules/openclaw/dist/auth-profiles-DDVivXkv.js
open -a "Visual Studio Code" /opt/homebrew/lib/node_modules/openclaw/dist/auth-profiles-DRjqKE3G.js
open -a "Visual Studio Code" /opt/homebrew/lib/node_modules/openclaw/dist/discord-CcCLMjHw.js
open -a "Visual Studio Code" /opt/homebrew/lib/node_modules/openclaw/dist/model-selection-46xMp11W.js
open -a "Visual Studio Code" /opt/homebrew/lib/node_modules/openclaw/dist/model-selection-CU2b7bN6.js
```

---

## 6. Why Exempting 198.18 Is Safe

198.18.0.0/15 is the RFC 2544 benchmark address range, virtually unused in real-world networks. Shadowrocket chose this range for fake-ip specifically because it does not conflict with real LAN addresses.

Shadowrocket will never use 127.x (loopback), 10.x, or 192.168.x (common LAN ranges) for fake-ip, as these would conflict with local network services. Therefore, exempting only 198.18 does not weaken SSRF protection against other private addresses.

---

## 7. Version Upgrade Notes

When OpenClaw is upgraded, the `dist` directory is overwritten and all modifications are lost. After upgrading:

1. Find all copies in the new version:

```bash
grep -rn "skipPrivateNetworkChecks) assertAllowedResolvedAddressesOrThrow" /opt/homebrew/lib/node_modules/openclaw/dist/*.js
```

2. Filenames and line numbers may change (filenames contain hashes), but the fix is identical
3. Apply the same modification to each file found (comment original + add new line)
4. Run `openclaw gateway restart`

If OpenClaw officially merges PR #24427 (`tools.web.fetch.allowPrivateNetwork` config option), you can switch to the config-based solution and stop modifying source code. Check PR status at: https://github.com/openclaw/openclaw/pull/24427

Related issue: https://github.com/openclaw/openclaw/issues/29669

---

## 8. Verification

1. Run `openclaw gateway restart`
2. Have the agent use `web_fetch` on any external URL, e.g. `https://github.com/openclaw/openclaw`
3. If page content is returned instead of an error, the fix is working
4. If it still fails, use the grep command above to check for unmodified copies
