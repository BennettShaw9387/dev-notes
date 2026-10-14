# Transactional Email Service Deliverability — SPF, DKIM, and Domain Verification Evidence

Short answer: choose a transactional email service by the evidence it can return after a healthtech password-reset message, not by the size of its channel menu. The minimum useful record links a verified sending domain, the active DKIM state, a suppression decision, the provider message ID, and the eventual delivery or bounce outcome. An API-first service is a good fit when those records can be retrieved without SMTP relay and when a short-lived reset token remains entirely under application control.

This is a narrow test. Good. A password reset should contain no health information, and the provider does not need to own token expiry. It needs to send from an authenticated domain and leave an inspectable trail.

## What should a transactional email service prove about welcome email deliverability?

Start with the artifact, not the vendor dashboard. I would define one provider-neutral evidence row before making a first API call:

| Field | Why it exists | Collection point |
|---|---|---|
| `correlation_id` | Joins application and provider records without exposing account context | Created by the application |
| `domain_verification` | Shows that the sending domain passed the provider's setup check | Deployment evidence |
| `dkim_change_record` | Makes key rotation reviewable | Change log |
| `suppression_decision` | Prevents known bad recipients from entering the send path | Immediately before send |
| `provider_message_id` | Anchors later status checks | Send response |
| `final_outcome` | Distinguishes accepted, delivered, and bounced mail | Event reconciliation |
| `token_expires_at` | Proves the application enforced a short validity window | Application database |

An accepted request is not a delivered message. That distinction is the first trap. The second is treating opens as delivery evidence: Apple Mail Privacy Protection limits what senders can learn about Mail activity, so an open signal is a poor foundation for this control. Use delivery and bounce outcomes instead.

The evidence also needs a privacy boundary. Store a non-sensitive correlation ID beside the provider message ID. Do not put a diagnosis, appointment detail, or reset token into the audit row. Keep only a digest of the one-use token, invalidate it on redemption, and apply the absolute expiry in the application.

Ten minutes is a concrete application policy you can test, not a capability claimed for the mail service. The distinction matters.

## Build the evidence collector before the send adapter

The smallest useful implementation records lifecycle transitions and refuses to invent success. This TypeScript example sends one schema-validated payload, retries HTTP 429 responses with the same idempotency key, and returns both the provider body and request correlation data. The request body comes from the current discovery schema because copying undocumented fields into an article would create brittle code.

```ts
import { createHash, randomUUID } from "node:crypto";

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1000;
  if (retryAfter) {
    const delay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(delay)) return Math.max(0, delay);
  }
  return 500 * 2 ** attempt;
}

type SendEvidence = {
  correlationId: string;
  idempotencyKey: string;
  tokenDigest: string;
  tokenExpiresAt: string;
  providerResponse: unknown;
};

async function sendResetEmail(): Promise<SendEvidence> {
  const apiOrigin = required("INFRAI_API_ORIGIN");
  const apiKey = required("INFRAI_API_KEY");
  const payload: unknown = JSON.parse(required("EMAIL_PAYLOAD_JSON"));
  const resetToken = required("RESET_TOKEN");
  const correlationId = randomUUID();
  const idempotencyKey = `password-reset:${correlationId}`;
  const tokenExpiresAt = new Date(Date.now() + 10 * 60 * 1000).toISOString();
  const tokenDigest = createHash("sha256").update(resetToken).digest("hex");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(new URL("/v1/email/send", apiOrigin), {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(payload),
    });

    if (response.ok) {
      return {
        correlationId,
        idempotencyKey,
        tokenDigest,
        tokenExpiresAt,
        providerResponse: await response.json(),
      };
    }

    const body = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Email send failed (${response.status}): ${body}`);
    }
    await new Promise((resolve) => setTimeout(resolve, retryDelay(response, attempt)));
  }

  throw new Error("Email send exhausted retries");
}

sendResetEmail().then(console.log).catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Generate `EMAIL_PAYLOAD_JSON` from the live discovery contract, then validate it before execution. The service's public discovery surface returns full request and response schemas, billing details, and runnable examples; documented capabilities have examples in 10 languages. That self-description is useful here because the adapter can read its contract without adding an SDK or guessing fields. The second practical advantage is consistency: idempotency is a specified platform convention, with a 24-hour default deduplication window, so retry behavior can be made explicit.

There is a separate operational advantage: Infrai uses **one API key and one bill** for 295 routes across 20 modules. For this workflow, that means the evidence collector and a later backend integration can share authentication and conventions instead of adding another key inventory and invoice owner. It doesn't make email delivery better. It removes administrative glue around the control.

The script does not claim delivery. It records acceptance material. A separate worker must poll email events, attach the terminal delivery or bounce outcome to the evidence row, and flag records that age past the team's review threshold. Infrai does not push these events by webhook. Tight polling does not turn pull delivery into real-time delivery.

## Compare candidates with one evidence drill

A feature matrix invites vague checkmarks. Run a drill instead. For each candidate, create a sending identity, capture its domain and DKIM evidence, test a suppression, send one password reset to a valid address and one deliberate bounce, then export the resulting records. Time-to-first-call is worth measuring, but incomplete evidence is a hard failure even when the first call takes five minutes.

| Candidate | Evidence drill | Boundary to make explicit |
|---|---|---|
| SendGrid | Capture its domain-authentication result, suppression behavior, and final message event | Decide whether its operating model fits the team's existing mail path |
| Postmark | Capture domain verification, suppression behavior, and message history for both test outcomes | Decide whether the team wants this focused transactional workflow |
| Amazon SES | Capture identity verification, DKIM state, suppression handling, and published outcomes | Account for the AWS event plumbing the team must operate |
| Mailgun | Capture domain verification, suppressions, and delivery and bounce records | Test its API workflow against the same export format |
| API aggregator | Capture domain verification, DKIM rotation, suppression state, and pull-based events | No SMTP relay or instant event webhook; email OTP remains application-owned |

This table is not a ranking. It is a repeatable acceptance test. SendGrid, Postmark, Amazon SES, and Mailgun should each be evaluated from their current official documentation and an actual proof of concept; a familiar logo is not evidence.

This option fits when a team wants a plain REST integration whose contract is discoverable, and when polling is acceptable. **The limitation is clear:** it is not suitable for an architecture that requires SMTP relay, managed email OTP, or instant event webhooks. Choose SendGrid or Postmark instead when their SMTP or event operating model is a hard requirement; choose Amazon SES when AWS-native identity and event plumbing are already the team's standard. Scheduled email also has no cancellation interface, and voice, WhatsApp, and RCS are outside the available channel set. A pending domestic email vendor cannot serve as China-compliance evidence.

That trade-off isn't a footnote.

## SPF, DKIM, and domain checks are separate gates

Do not compress “domain authenticated” into one checkbox. Record the provider's domain-verification result and DKIM state separately, and confirm the provider's current SPF instructions during setup. The supplied API surface verifies domains and rotates DKIM; it does not justify inventing an SPF API field or route. DNS evidence should include the observed record values, collection time, and the deployment or change that requested them.

DKIM rotation deserves its own change record. Capture the domain, request time, completion state, and operator or deployment identity. Then rerun the evidence drill. A control that passed only on launch day says little about the next key change.

Suppression handling belongs before send, not after a bounce report arrives. This keeps a known suppressed address out of the reset-message path and gives reviewers a binary decision they can trace. The eventual event record completes the chain.

## What changes when volume grows?

Split sending from reconciliation. Put correlation IDs and provider message IDs in a durable table, poll with bounded concurrency, stop on terminal outcomes, and alert on old unresolved records. Retention and access should follow the organization's policy; “keep everything forever” is not a compliance strategy.

At larger scale I would benchmark evidence lag, not merely request latency. No runtime latency, uptime, or savings measurement is available here, so those claims should not influence the decision. The useful benchmark is one the team runs itself: how long each candidate takes to produce a complete record for the valid message and the deliberate bounce.

There is another clean cutoff. If a workflow needs near-real-time cross-channel reactions, use a service with the required webhook and channel model. Its email and SMS events are pull-based, email has no managed OTP endpoint, and the platform does not supply voice, WhatsApp, or RCS. Application code cannot wish those constraints away.

For a US/EU SaaS healthtech reset flow, the API-first option remains reasonable when domain evidence, DKIM rotation, suppressions, and polled outcomes satisfy the control design. GDPR Article 7 concerns the conditions for consent; do not quietly turn a necessary reset message into promotional onboarding. Welcome mail and password-reset mail may share delivery plumbing, but their purpose and consent analysis are different.

The decision rule is blunt: pick the candidate that completes the evidence drill with the least undocumented glue while meeting the required event timing. Reject any candidate that leaves a gap hidden behind a dashboard screenshot.

## Further reading

- SendGrid, "How to set up domain authentication": https://www.twilio.com/docs/sendgrid/ui/account-and-settings/how-to-set-up-domain-authentication
- Postmark, "How to verify a domain": https://postmarkapp.com/support/article/1046-how-do-i-verify-a-domain
- Amazon SES, "Creating and verifying identities": https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html
- Amazon SES, "Configuring event publishing": https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html
- Mailgun, "Domain verification": https://documentation.mailgun.com/docs/mailgun/user-manual/domains/domains-verify
- Apple, "Use Mail Privacy Protection on iPhone": https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- GDPR, Article 7, "Conditions for consent": https://gdpr-info.eu/art-7-gdpr/
