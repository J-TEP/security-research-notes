# Testing a GraphQL API for Web Cache Deception and Broken Access Control

**Type:** Methodology note (authorized bug bounty testing)
**Target:** A retail e-commerce site's GraphQL backend-for-frontend (BFF) API, tested under a public bug bounty program with safe harbor
**Tools:** Burp Suite (Proxy, Repeater, Match and Replace), Chromium
**Outcome:** No vulnerability found. The API's caching and access controls were correctly configured.

> The target is anonymized and endpoint details, hashes and tokens are omitted, in line with the program's disclosure policy. All testing used my own two test accounts registered with my researcher email alias.

---

## 1. Setup

- Configured a Burp **Match and Replace** rule to append a researcher identifier to the `User-Agent` header, as the program requested, and verified it in the auto-modified requests.
- Added the target domain to Burp's **scope** and filtered HTTP history to in-scope traffic only.
- Created two test accounts (A and B) using researcher email aliases.

## 2. Mapping the API

- Browsed the account pages while proxying traffic and found that personal data loaded from a single GraphQL endpoint rather than from the page HTML.
- Catalogued the operation names from the `X-Operation-Name` header and request bodies. They included user info, addresses, email preferences, loyalty data and navigation queries.
- Noticed the client sent **full query text**, not only persisted query hashes, so the server accepted arbitrary queries.

## 3. Hypothesis: Web Cache Deception

Response headers showed the site sat behind a CDN with three cache layers (`X-Cache: MISS, HIT, MISS`).

| Observation | Cache-Control | X-Cache |
|---|---|---|
| Account page HTML | `max-age=0, no-store` | MISS ×3 |
| GraphQL **GET** (generic page text) | `public, max-age=0` | HIT (346 hits) |
| GraphQL **POST** (user info + session token) | `max-age=0, no-store` | MISS ×3 |

**Hypothesis:** If cacheability were set by HTTP method rather than by content, a sensitive operation sent as a GET might inherit `public` and be cached, exposing one user's data, including a session token, to others.

## 4. Testing

| Test | Method | Result |
|---|---|---|
| Introspection | `__schema` query | Blocked (`INTROSPECTION_DISABLED`). Backend identified as Apollo Server |
| Field suggestions | Misspelled a known field | Enabled (`Did you mean "textFor"?`). Minor hardening issue |
| Sensitive op over GET | Moved the user-info query into URL params | Executed (200) but returned `no-store`, MISS ×3 |
| Auth requirement | Same GET with all cookies removed | All personal fields returned `null`; server issued a new anonymous session |

## 5. Conclusion

- **Cache deception: not vulnerable.** The BFF marks responses by content (`Bff-Api-Response: user-profile-get-200` → `no-store`), not by HTTP method. Generic text is cacheable; personal data is not, even over GET.
- **Unauthenticated access: not vulnerable.** Personal fields resolve to `null` without a valid session.
- **Hardening note:** GraphQL field suggestions remain enabled in production, which lets someone partially map the schema even with introspection disabled.

## 6. Lessons Learned

- Check `Cache-Control` per operation, not per endpoint. One endpoint can serve both cacheable and non-cacheable responses.
- A response containing sensitive data is only a vulnerability if the wrong party can get it: without auth, through a shared cache, or across accounts.
- Disabling introspection doesn't fully hide a schema if field suggestions are still on.
- A clean negative result, documented with evidence, is still useful security work.
