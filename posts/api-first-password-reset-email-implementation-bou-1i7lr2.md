# API-First Password Reset Email Implementation — Bounce Suppression for Logistics

TL;DR: put password reset email behind a persistent job and check a local suppression record immediately before dispatch. For a logistics account, that is the least complex API-first implementation that prevents a known-dead address from receiving repeated attempts while keeping the web request fast. Treat the delivery API response as acceptance, not proof that the mailbox received anything.

| Choice | Request path | Bounce handling | Best fit |
|---|---|---|---|
| Persistent job plus local suppression | Commit job, then return | Events update one local recipient state | Operational accounts and retrying workers |
| Direct API call plus remote suppression | Call delivery API, then return | Delivery service owns most recipient state | Small systems with low retry complexity |
| SMS fallback | Separate channel and consent path | Its own delivery state | Existing, approved phone recovery flows |

**Recommendation:** choose the persistent job and local suppression ledger. It avoids an SMTP relay without pretending a remote API call and a local database write are one transaction. The runner-up, a direct call, stays simpler only while the application can accept weak recovery semantics around timeouts and process exits.

## How should an API-first password reset email implementation handle bounces?

A reset endpoint has two audiences. The user wants a quick, non-revealing response. Operations needs to know whether a delivery attempt should exist at all. Joining those concerns inside one Node.js HTTP request creates an awkward failure boundary: the delivery service may accept the request just as the application loses the response, or the application may retry after its own timeout.

That ambiguity matters in logistics. A dispatcher can request another link while a driver is waiting at a gate, then do it again from a handheld device. If the address is already known to be invalid, every new attempt adds noise without improving recovery. The useful unit of work is not “send an email.” It is “create one eligible recovery notification, then record what happened.”

SPF does not solve this. RFC 7208 describes authorization for a host to use a domain in a mail identity. It can support sender authentication, but it does not certify that a recipient mailbox exists or that a specific message arrived. Recipient state still belongs in the application delivery workflow.

Keep the public response boring. Return the same result for an unknown account, a suppressed address, and an eligible address. Internally, those paths should produce different, queryable outcomes.

No leak. No guesswork.

## Criterion one: can the system explain one attempt?

Start with an identifier that survives every handoff. One reset request gets an internal notification ID. The queued job carries it. The delivery adapter attaches it as metadata when its API supports metadata. An incoming event maps back to it. Logs help during debugging, but logs are not the state machine. The minimum useful model has 6 states: `queued`, `submitted`, `delivered`, `temporarily_failed`, `permanently_failed`, and `suppressed`. These are application states, not claims about a service's event vocabulary. Normalize external events at the adapter boundary and store unknown events for inspection; never let them silently change recipient eligibility. Then benchmark 3 spans separately: database commit to worker pickup, worker pickup to API acknowledgement, and acknowledgement to the latest delivery event. A single end-to-end average hides queue delay. Percentiles and sample counts matter more than a pretty average, but do not invent a service objective before collecting traffic from this flow. Keep 2 timestamps on every event: when the external system says it occurred and when the application received it. Consider a job submitted at 10:00, a permanent-failure event recorded externally at 10:01, and an acknowledgement that reaches the application at 10:02. Processing by arrival order would move the notice backward to `submitted`, even though the stronger recipient outcome already exists. A precedence rule rejects that regression; an event ID makes the repeated failure harmless. That concrete ordering belongs in a fixture, not in an operator's memory.

**One notification ID must explain one decision, one submission, and every later state transition.** If a design cannot do that without searching several dashboards, its short setup time is misleading.

## Criterion two: who owns recipient eligibility?

A delivery platform may suppress recipients within its own account, but the application still decides whether a password reset job is eligible. A local ledger makes that decision visible before any network call. It also keeps the rule stable if transactional mail is later split across regions, accounts, or adapters.

The ledger needs a normalized address key, status, reason category, source event ID, effective time, and last update time. Avoid copying raw event bodies into every business table. Retain the original event once under the applicable security and retention policy, then project only the fields the dispatcher needs.

Not every failed attempt should permanently suppress an address. A temporary delivery failure belongs on a bounded retry path. A permanent recipient failure can make the address ineligible until it is corrected or explicitly revalidated. Those categories must come from the adapter's documented mapping. Free-form text matching will classify the wrong thing.

There is an important asymmetry. A suppression update may cancel a queued notification. A new reset request must never clear suppression. Recovery from suppression needs a separate, authenticated business action, such as changing and verifying the account address. The trade-off is explicit: this can delay recovery for someone whose address was corrected outside the application, but automatically clearing the record would let repeated reset requests override delivery evidence. I would choose the explicit revalidation path because its state transition can be audited.

**Run the final eligibility check in the worker, immediately before submission.** Checking only when the job is created leaves a race between a later bounce event and a delayed job.

## What does the TypeScript boundary look like?

Express and Next.js can call the same application service. Keep token creation, job persistence, and mail transport behind interfaces so a route does not grow delivery-specific conditionals. Persist the reset record and notification job together. Do not write the secret-bearing reset token to logs or delivery metadata.

```ts
type RecipientStatus = "eligible" | "suppressed";
type NoticeState =
  | "queued"
  | "submitted"
  | "delivered"
  | "temporarily_failed"
  | "permanently_failed"
  | "suppressed";

type ResetNotice = {
  id: string;
  recipientKey: string;
  resetUrl: string;
  state: NoticeState;
};

interface NoticeStore {
  claimNext(): Promise<ResetNotice | null>;
  recipientStatus(key: string): Promise<RecipientStatus>;
  markSuppressed(id: string): Promise<void>;
  markSubmitted(id: string, externalId: string): Promise<void>;
  releaseForRetry(id: string): Promise<void>;
}

interface DeliveryClient {
  submit(input: {
    notificationId: string;
    toKey: string;
    template: "password-reset";
    variables: { resetUrl: string };
  }): Promise<{ externalId: string }>;
}

async function dispatchOne(
  store: NoticeStore,
  delivery: DeliveryClient
): Promise<void> {
  const notice = await store.claimNext();
  if (!notice) return;

  if (await store.recipientStatus(notice.recipientKey) === "suppressed") {
    await store.markSuppressed(notice.id);
    return;
  }

  try {
    const result = await delivery.submit({
      notificationId: notice.id,
      toKey: notice.recipientKey,
      template: "password-reset",
      variables: { resetUrl: notice.resetUrl }
    });
    await store.markSubmitted(notice.id, result.externalId);
  } catch {
    await store.releaseForRetry(notice.id);
  }
}
```

Retry counts and timing are absent on purpose. They are policy, not universal constants. Set them from token lifetime, measured queue latency, and the adapter's documented retry guidance. A retry after token expiry is successful transport work with zero user value.

The catch block exposes a harder trade-off: submission can succeed while `markSubmitted` fails. Production code needs an idempotency mechanism at the delivery boundary when available, or reconciliation keyed by notification ID. One database and one remote API do not form a transaction. Ignoring that gap is how duplicate reset messages appear.

Verify incoming event authenticity with the selected service's documented method before changing state. Deduplicate by external event ID, preserve occurrence time, and reject transitions that move a terminal permanent failure back to submitted. Test the ugly orderings: permanent failure before acknowledgement, duplicate delivery, an event for an unknown notification, and suppression arriving after worker claim but before submission.

5 fixtures beat 50 lines of framework glue.

Stop there.

## When is the direct call the better runner-up?

A direct API call can be the honest choice for an early internal tool when there is no worker, no automatic request retry, and someone can inspect every failure. It removes a queue and gets to the first call quickly. Keep the delivery client interface anyway. The adapter is cheap; leaking transport response shapes through route handlers is expensive later.

The direct path stops being simple when the endpoint must wait on a remote network, when users repeat requests after ambiguous responses, or when several applications send recovery mail. At that point, remote suppression alone cannot explain the application's decision before the call. The persistent job earns its extra table.

An SMTP relay does not remove this state problem. It changes the handoff, while the application still needs to correlate later recipient outcomes with a recovery attempt. Pick HTTP API or SMTP based on the team's operational boundary, then apply the same suppression and evidence rules.

SMS is not a drop-in runner-up for an email bounce. An SMS API has its own message resources and delivery behavior. Use it only when the product already has a verified phone recovery policy, consent handling, and channel-specific status processing. Do not silently switch channels because an email address bounced.

Before shipping, test every state transition with non-production recipients. Confirm that duplicate events remain idempotent, suppressed jobs make no network call, unknown events are retained without changing eligibility, and an expired reset cannot be dispatched. Then benchmark the three spans under planned worker concurrency. Those knobs answer operational questions. Everything else can wait.

Persist recovery notifications, own the suppression decision locally, and make the final check at dispatch time. That design adds one durable boundary, but it provides the evidence needed to handle invalid logistics recipients without turning password recovery into repeated guesswork.

## References

- RFC 7208, Sender Policy Framework (SPF): https://datatracker.ietf.org/doc/html/rfc7208
- Twilio SMS official documentation: https://www.twilio.com/docs/sms
