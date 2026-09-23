# How to Build Magic Link Email Codes — Verified Sessions and Audit Logs

A magic link email flow should create a session only after a single-use, short-lived code is verified, while recording deliverability and verification as separate events. For a developer tool migrating away from managed authentication, that separation is the deciding constraint: an accepted email is not proof of delivery, and delivery is not proof of identity.

TL;DR: generate the secret with a cryptographically secure random source, store only a keyed digest, bind it to one normalized account identifier and one purpose, consume it atomically, rotate the session identifier after verification, and keep provider messages out of the security state machine. Benchmark the whole path from request to usable session. The smallest useful measurements are send-accept latency, delivery latency, verification success, expiration, and duplicate webhook rate.

## How should a Node.js magic link verify an email code?

The login boundary has three jobs. It must keep the emailed secret unpredictable, prevent replay, and avoid revealing whether an address has an account. OWASP recommends generic authentication responses because different messages and response timing can enable account enumeration. The Node.js request handler should therefore return the same public response for known and unknown addresses.

Keep that response boring.

Email introduces a second boundary. A mail API accepting a message means the request reached that API; it does not establish inbox placement. Record the provider message identifier as correlation data, then update delivery state from authenticated webhooks. Never let a delivery callback create a browser session.

For a CLI or SDK business, this matters during migration. Existing users may arrive with mixed-case addresses, old session cookies, and provider-specific identifiers. Pick one canonical email normalization rule before import, preserve the old subject as migration metadata, and make the new internal user ID the session subject. Do not couple authorization to an email address that a user may later change.

## Build the smallest verifiable path

This TypeScript example uses a 6-digit email code because it is easy to enter in a terminal-driven device flow. Those 6 digits provide only 1,000,000 possible values, so the code is protected by a 5-minute lifetime, a 5-attempt ceiling, one-time consumption, and request throttling outside this function. A link carrying a longer random token can use the same persistence and transaction shape, but a short code needs all of these compensating controls. The trade-off is less friction at the keyboard in exchange for stricter throttling and expiry. Those controls are part of the design.

```ts
import { createHmac, randomInt, randomUUID } from "node:crypto";

type Challenge = {
  id: string;
  email: string;
  digest: string;
  expiresAt: Date;
  attempts: number;
  consumedAt: Date | null;
};

type AuditEvent = {
  kind: "code_requested" | "email_accepted" | "email_delivered" | "verify_failed" | "session_created";
  challengeId: string;
  at: Date;
  detail?: Record<string, string>;
};

interface Store {
  insertChallenge(value: Challenge): Promise<void>;
  appendAudit(value: AuditEvent): Promise<void>;
  consumeAndCreateSession(input: {
    challengeId: string;
    digest: string;
    now: Date;
    maxAttempts: number;
    sessionId: string;
  }): Promise<{ ok: boolean; sessionId?: string }>;
}

interface Mailer {
  sendCode(input: { to: string; code: string; challengeId: string }): Promise<{ messageId: string }>;
}

const CODE_TTL_MS = 5 * 60 * 1000;
const MAX_ATTEMPTS = 5;
const secret = process.env.AUTH_CODE_HMAC_KEY;
if (!secret) throw new Error("AUTH_CODE_HMAC_KEY is required");

function normalizeEmail(value: string): string {
  return value.trim().toLowerCase();
}

function digest(challengeId: string, email: string, code: string): string {
  return createHmac("sha256", secret)
    .update(`${challengeId}\n${email}\n${code}`)
    .digest("hex");
}

export async function requestCode(store: Store, mailer: Mailer, rawEmail: string): Promise<void> {
  const email = normalizeEmail(rawEmail);
  const id = randomUUID();
  const code = randomInt(0, 1_000_000).toString().padStart(6, "0");
  const now = new Date();

  await store.insertChallenge({
    id,
    email,
    digest: digest(id, email, code),
    expiresAt: new Date(now.getTime() + CODE_TTL_MS),
    attempts: 0,
    consumedAt: null
  });
  await store.appendAudit({ kind: "code_requested", challengeId: id, at: now });

  const sent = await mailer.sendCode({ to: email, code, challengeId: id });
  await store.appendAudit({
    kind: "email_accepted",
    challengeId: id,
    at: new Date(),
    detail: { messageId: sent.messageId }
  });
}

export async function verifyCode(
  store: Store,
  input: { challengeId: string; email: string; code: string }
): Promise<{ ok: boolean; sessionId?: string }> {
  const email = normalizeEmail(input.email);
  return store.consumeAndCreateSession({
    challengeId: input.challengeId,
    digest: digest(input.challengeId, email, input.code),
    now: new Date(),
    maxAttempts: MAX_ATTEMPTS,
    sessionId: randomUUID()
  });
}
```

One transaction. No exceptions.

The database operation behind `consumeAndCreateSession` compares the digest, checks that the challenge is unconsumed and unexpired, increments failed attempts, marks a successful challenge consumed, and inserts the session. A read followed by a later update leaves a replay window when two verification requests race. Consider the concrete failure: two requests read the same unused challenge, both verify the same digest, and both create sessions before either update becomes visible. Putting the conditional update and session insert in one transaction makes the database decide which request wins. The losing request gets the same generic verification failure used for an expired or already consumed challenge; it does not receive a clue about the account state.

The HMAC key belongs in a secret manager and needs a rotation plan. Store a key version beside each challenge if challenges must survive rotation. Compare digests in the database or with a constant-time comparison in application code; ordinary string equality can leak timing information.

## Make delivery observable without granting trust

Treat the audit log as append-only security evidence. Keep the challenge ID, event type, timestamp, provider message ID, and a coarse failure reason. Avoid storing the code, digest, full email body, cookie, or raw session token. Restrict access and define retention up front because an audit table full of identifiers becomes its own liability.

Webhook ingestion needs signature verification, replay validation where the sender supports it, and idempotency keyed by the provider event ID. Map accepted, delivered, bounced, and complained events into your internal vocabulary. Preserve the original event type in constrained metadata so an adapter change does not erase diagnostic detail.

Measure percentiles, not one average. The useful clock starts before the application calls the mail adapter and ends when verification succeeds. Split that duration into application queue time, provider acceptance, reported delivery, and user completion. This exposes the difference between a slow worker and a message that never reaches an inbox. It also makes deliverability an observed pipeline rather than a Boolean guessed from the send response. No invented target helps here: establish a baseline with production traffic, then set service objectives from user tolerance and observed behavior.

Don't log the secret.

The trap is logging everything because debugging feels urgent. A structured audit log event with a correlation ID usually answers the operational question without retaining authentication material.

## Test migration failures before switching traffic

Run the old and new identity mappings through an offline reconciliation first. Count duplicate normalized emails, missing verified-email evidence, users with multiple legacy subjects, and active sessions that cannot be revoked individually. Each category needs an explicit policy. Silent merging is especially dangerous because it can join identities that the previous system kept separate.

Then test the state machine with a fake clock and concurrent verification calls. Cover expiry at the boundary, the sixth attempt after five failures, two correct submissions at once, repeated delivery webhooks, an unknown email request, mail-adapter timeout, and session revocation. Assert that exactly one racing request receives a session. Assert the public response for an unknown address has the same shape as the response for a known one.

During rollout, route a small cohort through the new flow and compare completion and delivery telemetry by domain class without exposing individual addresses. Keep rollback at the adapter and traffic-routing layers. Database changes should be additive until old sessions have expired or been revoked. This is slower than a one-shot import, but it buys a clean reversal path when identity reconciliation exposes a bad assumption.

Migration rewards patience.

## What I would change at scale

I would move mail sending to a durable queue, put challenge creation and an outbox record in the same transaction, and let a worker deliver from that outbox. This adds a table, a worker, and retry policy. The config cost is justified only when a process crash between storing the challenge and sending the email is no longer an acceptable failure mode.

I would also replace a single global throttle with layered limits on account, network, device, and destination patterns. Limits must return the same generic response, or the anti-abuse control recreates enumeration. Tune them from observed abuse and false positives; a copied threshold is cargo cult security.

Finally, session storage should support server-side revocation, idle and absolute expiry, and rotation after authentication. Cookies carrying session identifiers need `Secure`, `HttpOnly`, and an appropriate `SameSite` policy. OWASP recommends renewing the session identifier after a privilege-level change, which includes authentication.

The trade-off is plain: a managed provider hides much of this machinery, while owning the flow gives control over identity mapping, audit retention, and delivery adapters. Migration is sensible only if the team is prepared to operate the state machine and its abuse controls. The winning implementation is the one whose failure states are explicit, testable, and reversible.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
- https://nodejs.org/api/crypto.html
- https://www.rfc-editor.org/rfc/rfc2104
