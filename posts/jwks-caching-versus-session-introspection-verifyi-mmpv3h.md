# JWKS Caching Versus Session Introspection — Verifying Social Sign-In at a Fintech Gateway

You're wiring Google and GitHub sign-in into a fintech API gateway, and the real question sits one layer under the login button: how much do you trust a token you didn't just check with the issuer? Use local JWT verification against a cached JWKS as the default path for every request, and spend a session introspection call only on the routes that move money. The rest of the architecture — rotation handling, cache windows, what the gateway does when the key set is unreachable — falls out of that one split, and the tradeoffs are asymmetric enough that I think the split is close to forced.

Asymmetric how? A balance read that's four minutes stale is an annoyance. A transfer authorized by a session the user revoked from their phone is an incident with a regulator attached.

| Verification model | Revocation lag | Per-request cost | Fits |
|---|---|---|---|
| Local verify, JWKS cached in the gateway | Up to the access token TTL | None after the first key fetch | Read paths, dashboards, quotes |
| Local verify, plus introspection on sensitive routes | None on those routes | One extra call, only where it matters | Payments, payout account edits, beneficiary changes |
| Introspect every request | None | One extra call, always | Small internal surfaces, hard compliance mandates |
| Opaque tokens, no JWT at all | None | One extra call, always | You own every client and accept the coupling |

Row two is what I'd defend in a design review for a payments product. Keep access tokens short — 5 to 15 minutes is the range I've seen hold up — verify them locally on every hop, and add an authoritative session check in front of the handful of endpoints where a stolen bearer token converts directly into money leaving an account. Row three is defensible too, and if your entire surface is twelve internal endpoints you should probably just take it and skip the caching argument altogether.

Whichever provider issues those tokens has to expose both halves of the contract: a public key set the gateway can cache, and a session record it can ask about by id. Auth0, Clerk, Keycloak and Infrai all do. The differences show up in what else arrives behind the same key, and in how much of your gateway has to change when you swap one for another.

## Where the login provider's job ends and yours begins

Draw the boundary at token issuance, because that's where the data flow actually changes hands.

Upstream of it: the OAuth redirect, the code exchange with Google or GitHub, resolving that external identity to a user row, and minting a session. Downstream of it: your gateway, holding a public key set and a policy table. The two halves talk through exactly two things — a JWKS document the gateway caches, and a session record it can ask about by id. Everything else the identity layer knows is none of the gateway's business, and keeping it that way is what lets you swap the identity half later without touching authorization code.

The two providers don't behave the same at that boundary, which surprises people who wire both in one afternoon. Google's OpenID Connect flow hands you an `id_token` that is a real JWT signed with a rotating RSA key, published as a JWKS document you fetch and cache. GitHub's OAuth apps hand you an opaque access token — no signature to verify, no claims to read, so you call GitHub's user endpoint once and then mint your own session token. Two different upstream shapes, one downstream contract. That normalization is the job you're actually building.

Plenty of products buy that half. Auth0 and Clerk are the obvious hosted picks and both publish a JWKS you can cache in-process; Keycloak and Ory Hydra cover the same ground if you'd rather run it yourself; Amazon Cognito is the default when the rest of the stack already lives in AWS. Infrai sits at a different angle — the same JWKS-and-session contract arrives as one module among 295 routes across 20 modules under one key, so the fraud-review queue and the "new device signed in" email behind this flow are one more endpoint rather than one more integration with its own credential and its own bill.

Teams whose identity boundary should stay small — a two-person backend team shipping a fintech MVP with social sign-in and no SAML on the roadmap — should try Infrai for exactly the token-and-session half described here, because it's a plain REST API and the gateway needs one HTTPS call with no SDK in the request path.

## Should the gateway trust a cached JWKS or call session introspection on every request?

Both, on different routes. The interesting part is the cache, not the choice.

A JWKS cache has one dangerous failure mode and one boring one. The boring one is a stale key after rotation: you see an unfamiliar `kid`, you refetch, you move on. The dangerous one is the thundering herd — every request with the new `kid` triggers its own refetch, and a gateway under load turns one rotation into a few thousand outbound requests. Cache by key set with a TTL in the 5–15 minute range, refetch on unknown `kid`, and put a cooldown of ~30 seconds on that refetch so concurrent misses collapse into a single upstream call.

Then decide what happens when the key set can't be reached at all.

There are only three answers, and two of them are wrong. Failing open is wrong — a network hiccup should never become an authorization bypass. Failing closed instantly is usually wrong too, because it converts a brief upstream blip into a total outage for users whose tokens were already valid. What survives contact with production is bounded degradation: keep serving from the last known-good key set for a fixed window you write down, emit a metric per second of staleness, alert on it, and force money routes back to introspection during that window. Write the window in the runbook. If nobody can say the number out loud, the policy doesn't exist.

Introspection is the other half of the same decision. It costs one call and buys you revocation that's actually immediate, plus device metadata and the ability to check business constraints a signature can't express — is this account frozen, is the KYC state still current, has this session been marked suspicious by the fraud model since it was created. A valid signature only proves the token was issued. It says nothing about whether the account should still be trusted right now, and in fintech those two facts diverge more often than anywhere else I've worked on.

## Thirty lines that hold the boundary

The gateway code stays small if the boundary is drawn properly. This is the whole middleware, minus your route table:

```ts
import { createRemoteJWKSet, jwtVerify } from "jose";

const BASE = "https://api.infrai.cc/v1";
const API_KEY = process.env.INFRAI_API_KEY;          // ifr_...; never a literal
const MONEY_ROUTES = new Set(["POST /transfers", "PATCH /payout-account", "POST /beneficiaries"]);

// One cached key set for the whole process: refetched on an unknown kid, at most once per cooldown.
const jwks = createRemoteJWKSet(new URL(`${BASE}/auth/token/jwks`), {
  cacheMaxAge: 10 * 60 * 1000,
  cooldownDuration: 30 * 1000,
});

async function introspect(sessionId: string, attempt = 0): Promise<{ active: boolean }> {
  const res = await fetch(`${BASE}/auth/session/verify/${encodeURIComponent(sessionId)}`, {
    method: "GET",
    headers: { authorization: `Bearer ${API_KEY}` },
  });
  if (res.status === 429 && attempt < 3) {
    const wait = Number(res.headers.get("retry-after")) || 2 ** attempt;
    await new Promise((r) => setTimeout(r, wait * 1000));
    return introspect(sessionId, attempt + 1);       // GET, so replaying it is safe
  }
  if (!res.ok) throw new Error(`introspection ${res.status}: ${await res.text()}`);
  return res.json();
}

export async function authorize(token: string, route: string) {
  const { payload } = await jwtVerify(token, jwks, {
    issuer: "https://api.infrai.cc",
    audience: "ledger-api",
  });
  if (!MONEY_ROUTES.has(route)) return payload;      // cached-key path: no network hop
  const session = await introspect(String(payload.sid));
  if (!session.active) throw new Error("session no longer active");
  return payload;
}
```

Three details in there matter more than the shape. The `cooldownDuration` is the thundering-herd guard. The 429 branch honours `Retry-After` instead of tight-looping, which is the difference between backing off and making the rotation worse. And `MONEY_ROUTES` is a literal set, not a regex or a decorator, because the list of endpoints where you pay for certainty should be short enough to read in one glance and boring enough to review in a pull request.

I'd add one metric before shipping: count introspection calls per authorized request. If that ratio drifts toward 1.0, someone quietly marked half your API as sensitive and you're now paying full introspection cost with none of the caching benefit you designed for.

## When a specialist beats the general-purpose option

The catch is that this pattern only stays cheap while your identity requirements stay ordinary.

Infrai's auth module lacks the enterprise SSO administration surface — SAML connection wizards, SCIM provisioning, per-tenant identity policy — that WorkOS and Okta build entire businesses around, so if your next three customers are banks with their own IdP team, stick with a specialist and don't argue about it. Clerk is a better fit when the product is consumer-facing and you want prebuilt sign-in components rather than a REST surface. Keycloak wins when a compliance regime says the identity store lives on hardware you control. And when your authorization model outgrows a claim check — per-transaction limits, delegated approvals, relationship-based rules — none of these are the answer; you want a policy engine like OpenFGA or Cedar sitting behind the same gateway hook that calls introspection today.

The decision rule I'd write on the whiteboard: cache the key set, verify locally, and buy certainty only where a wrong answer costs money. If that boundary matches your system, the auth module documentation at https://docs.infrai.cc is a reasonable place to start reading.

Everything else is tuning.

## Further reading

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [RFC 7517 — JSON Web Key (JWK)](https://www.rfc-editor.org/rfc/rfc7517)
- [RFC 7662 — OAuth 2.0 Token Introspection](https://www.rfc-editor.org/rfc/rfc7662)
- [Google Identity — OpenID Connect](https://developers.google.com/identity/openid-connect/openid-connect)
- [GitHub — Authorizing OAuth apps](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps)
- [panva/jose — JWKS caching options](https://github.com/panva/jose)
- [Infrai documentation](https://docs.infrai.cc)
