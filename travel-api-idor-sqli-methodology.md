# Testing a Travel/Loyalty Web API for IDOR, Mass Assignment, Injection, and Access Control

**Type:** Methodology note (authorized bug bounty testing, anonymized)
**Target:** A large travel booking platform's account & loyalty API, tested under a public bug bounty program with safe harbor
**Tools:** Burp Suite Community (Proxy, Repeater), Chromium
**Outcome:** No vulnerability found. The in-scope surface was consistently well-built: identity bound to the session cookie, sensitive values signed, queries parameterized.

> Target name, endpoints, tokens, and account identifiers are omitted in line with the program's disclosure policy. All testing used two of my own test accounts registered with my researcher email alias, and the required researcher identifier header.

---

## 1. Setup
- Added the researcher identifier to the `User-Agent` via a Burp Match-and-Replace rule, as the program required.
- Scoped Burp to the target host; confirmed no automated scanning (Community edition, manual Repeater only, per program rules).
- Registered two test accounts (A and B) to enable access-control testing between them.

## 2. Endpoints mapped
By clicking through the account, loyalty wallet, and trip features with the proxy on, I catalogued the data APIs: a wallet/cashback list, a profile name-change, a trip-list fetch, a saved-traveller fetch, and the booking setup call.

## 3. Tests run and results

| Class | Method | Result |
|---|---|---|
| **IDOR** | Swapped a user-GUID present in the saved-traveller request body to account B's GUID | Clean — server resolved identity from the session cookie and ignored the body value |
| **IDOR** | Checked wallet-list and trip-list endpoints for any user identifier to swap | Clean — no client-supplied ID; cookie-bound |
| **Mass assignment** | Injected extra fields (`balance`, `tier`, `isVip`, `verified`) into the profile name-change mutation | Clean — strict model bound only the expected fields; extras silently dropped, verified by reloading state |
| **Stored XSS** | Set the name field to an HTML-injection payload | Blocked — input validation rejected `<` |
| **Price tampering** | Inspected the booking-setup request for a client-controlled price | Not possible — price carried inside signed/encrypted tokens, no plaintext amount |
| **Input fuzzing** | Type confusion on numeric params (string/array where int expected) | Clean — graceful rejection, no stack trace or framework leak |
| **SQL injection** | Error-based (`'`, `' OR '1'='1`), boolean, and time-based (`WAITFOR DELAY`) on string and numeric params | Clean — no DB errors, no boolean change, no delay; type validation rejects bad numeric input pre-query |

## 4. Lone observation
One wallet endpoint accepted a **garbage anti-forgery token** and still returned data (identity came from the cookie). On a **read** endpoint behind a strict `Access-Control-Allow-Origin`, the impact is Informational/Low: an attacker can't read the cross-origin response and the request changes no state. Noted, not escalated.

## 5. Lessons
- A sensitive value appearing in a response (name, email, token) is only a vulnerability if the *wrong party* can obtain it: without auth, via a shared cache, or across accounts. Here, none applied.
- A client-supplied `userId` in a request body is an IDOR *candidate*, not an IDOR — test whether the server actually trusts it over the session.
- Clean, graceful rejection of bad input types is itself evidence of parameterized queries, which argues against SQLi before you even probe.
- A thorough **negative** result, documented end-to-end, is legitimate security work and demonstrates methodology.
