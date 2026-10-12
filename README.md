# Hacked AiFiesta

**Assessment Method:** Manual Testing + Proxy Interception + Client-side Inspection  
**Target:** aifiesta.ai Platform  

---

## 1. Executive Summary

During routine security testing of the platform, multiple **critical P0 vulnerabilities** were identified that allow:

-  Unlimited usage of premium Large Language Models (LLMs) without payment
-  Bypass of free-tier message limits (infinite usage)
-  Creation of unlimited chats & model executions
-  Misconfigured client-side access controls
-  Potential massive financial loss (LLM inference cost drain)

These issues exist due to:
- Client-side authorization logic
- Unverified model-access control
- Improper use of Supabase (anon keys & public endpoints)

**Immediate remediation is strongly recommended.**

---

## 2. Vulnerabilities Identified

###  VULNERABILITY #1 — Free Users Can Use Paid/Premium Models

**Severity:** Critical (P0)  
**Impact:** Full bypass of subscription system → direct financial loss

#### Description
The client sends a POST request to `/api/v2/completions` with the chosen model name. The server does not validate user subscription tier, resulting in:
- A free account directly selecting `model = claude`, `model = gpt-4`, `model = grok`, etc.
- Server processes the request normally
- Premium model responses are streamed back

#### Reproduction Steps
1. Create a free account
2. Open DevTools → Network
3. Start a conversation
4. Capture the POST request to `/api/v2/completions`
5. Edit JSON body:
```json
{
  "model": "claude-3-opus",
  "message": "Hello"
}
```
6. Resend the request with the free user's cookie
7. Premium model responds successfully

#### Impact
- Free users can exhaust expensive LLM credits
- Loss of revenue — subscription tier useless
- No server-side subscription enforcement

#### Recommended Fix
-  Enforce model-access authorization on the backend
-  Client-selected model must never be trusted
-  Reject model usage unless `user.subscription_tier >= model.required_tier`

---

###  VULNERABILITY #2 — Free Users Can Send Unlimited Messages

**Severity:** Critical (P0)  
**Impact:** Unlimited platform usage, resource exhaustion

#### Root Cause
Message count increment happens from the client via:
```
POST /message/count
```

Since the client controls when this request is sent, an attacker can:
1. Intercept request using Burp Suite
2. Drop the `/message/count` call
3. Still send LLM completion requests normally
4. Message count remains 0/3 forever → unlimited usage

#### Reproduction Steps
1. Sign into a free account
2. Open Burp Suite or browser request interceptor
3. Intercept outgoing requests
4. Drop all requests to: `POST /message/count`
5. Forward all other requests
6. Send unlimited prompts
7. Notice message counter never increments → unlimited usage

#### Impact
- Unlimited free usage
- Complete bypass of rate limits
- Potential denial-of-wallet (cost flooding)

#### Recommended Fix
-  Move message counting to backend
-  Only increment after verifying subscription tier and processing model request
-  Requests without increment MUST be rejected

---

###  VULNERABILITY #3 — Client-Generated Chat/Group IDs

**Severity:** High  
**Impact:** Cross-account chat injection, arbitrary chat creation, message spoofing

#### Findings
- Conversation IDs (UUIDs) are generated on client side
- Server trusts user-provided IDs without validation
- Multiple users can create chats with identical conversation IDs
- System does not enforce isolation per user ID

#### Recommended Fix
-  Generate chat IDs server-side
-  Enforce uniqueness + ownership mapping in DB
-  Reject any request where: `conversation.userId ≠ request.userId`

---

###  VULNERABILITY #4 — Supabase Public Anon Keys Exposed

**Severity:** High  
**Impact:** Potential data exposure

#### Findings
- Supabase anon key is sent to frontend
- Read/write operations are handled by client → unsafe
- JWT token is correctly verified, but schema permissions allow too much access
- Attackers can directly access DB endpoints using anon key unless RLS (Row Level Security) is perfectly configured

#### Recommended Fix
-  Move business logic (message count, chat creation, model routing) to backend server
-  Use Supabase service role keys ONLY on backend
-  Restrict anon key to only minimal operations

---

###  VULNERABILITY #5 — No Rate Limiting / Abuse Protection

**Severity:** High  
**Impact:** API credit drain, automated abuse, potential platform outage

#### Recommended Fix
Implement:
-  Global rate limiting (IP + user ID)
-  Billing-based hard limits
-  Abuse-detection monitoring

---

## 3. Overall Impact Analysis

| Vulnerability | Severity | Business Impact |
|--------------|----------|-----------------|
| Premium model access bypass | Critical | Direct financial loss (LLM credit burn) |
| Message limit bypass | Critical | Unlimited unmonitored usage |
| Client-generated IDs | High | Data leakage / spoofing risk |
| Supabase anon key misuse | High | Database exposure risk |
| No rate limiting | High | DoS, cost flooding |

---


## 4. Proof of Concept (PoC Snippets)

### Exploit 1 – Call Premium Model as Free User
```bash
curl https://api.aifiesta.ai/v2/completions \
  -H "Cookie: session=<FREE-USER-COOKIE>" \
  -d '{
        "model": "claude-3-opus",
        "message": "test"
      }'
```
**Result:** Claude returns full paid output.

### Exploit 2 – Unlimited Messages
1. Drop: `POST /message/count`
2. Then repeatedly call: `POST /api/v2/completions`

**Result:** Message counter never increases → unlimited usage.

---


---

**Please confirm receipt of this report and provide an estimated timeline for remediation.**

## Ethics

Use only on systems and content you are **authorized** to access. See [SECURITY.md](./SECURITY.md).

## License

Licensed under the [HackerX License](./LICENSE). Redistributions must retain copyright,
license text, and [NOTICE](./NOTICE) attribution per the license terms.
