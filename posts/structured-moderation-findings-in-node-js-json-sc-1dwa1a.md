# Structured Moderation Findings in Node.js: JSON Schema for Text, Image and Tenant Cost

Use one chat completions call with a strict JSON schema for both text and image moderation, then pick your provider on whether it hands back the cost of that call — an OpenAI-compatible API has no separate moderation route, so the safety check is a billable classification prompt firing on every submission, and in a multi-tenant product somebody's invoice has to absorb it.

| Option | How you call it | Per-call cost you can attribute | Where it fits |
| --- | --- | --- | --- |
| OpenAI moderation endpoint | dedicated `omni-moderation` route, text and images | nothing to attribute — that endpoint isn't billed | you already run on OpenAI and want a maintained classifier |
| Azure OpenAI + AI Content Safety | REST call returning a severity per category | per transaction, reconciled from Azure billing later | region-pinned deployments with an audit trail |
| OpenRouter | OpenAI-compatible chat across many vendors | fetched afterwards from its generation endpoint | trying several models before you commit |
| Ollama, self-hosted | local HTTP, open-weight vision models | no per-call price at all; you amortise hardware | volume high enough that per-call billing hurts |
| Infrai | OpenAI-compatible chat on the same key as its other REST modules | returned inline with the completion | teams that want the check and the rest of the backend on one contract |

Take a code review service for fintech teams: a diff goes in, structured findings come back out, and every tenant is a different bank with a different budget owner. The parts that need screening are the human bits around the diff — free-text context an outside contractor pastes in, a screenshot someone attaches to explain a failing case. If that check runs on every submission and you can't say which tenant caused which line of spend, your unit economics are a guess dressed up as a number.

That last column is the decision.

## How does a Node.js service check text and image safety on an OpenAI-compatible chat API?

The compatible surface gives you `/v1/chat/completions`. That's the toolbox. The moderation contract therefore has to live in the response format instead of in the endpoint: `response_format` of type `json_schema`, `strict: true`, and an enum your code can switch on without parsing prose.

Three labels cover most gates — allow, review, block — with a category array drawn from hate, sexual, violence, self-harm, harassment and spam. Keep the schema closed (`additionalProperties: false`) so a model that gets creative produces a validation error rather than a field your handler silently ignores.

Images ride the same call. Send an `image_url` content part next to the text part and ask for the identical schema back, so nothing downstream branches on modality. Not every chat model takes image input, so read `/v1/ai/models` first and pin an id that does — a junior on the team should be able to see which models are available in the US or EU region they're deploying to, rather than finding out from a support ticket.

If the response doesn't validate against the schema, treat it as review, never as allow. Fail closed.

## Per-tenant cost attribution beats token math

There are two ways to know what a tenant's safety checks cost you. The first is to reconstruct it: read `usage.prompt_tokens` and `usage.completion_tokens`, multiply by a price sheet you keep in your own config, and sum per tenant. It holds up until a router picks a different vendor for you, or an image gets tokenised by tiles at a rate your sheet doesn't model, or a cached prefix changes what you were actually charged — at which point your ledger and the invoice disagree, and you're the one explaining the gap to finance with a spreadsheet you wrote at midnight.

The second way is to let the API tell you. Infrai's OpenAI-compatible responses carry a top-level `infrai` object — `cost_usd`, `vendor`, `model`, `request_id` — plus `X-Infrai-Cost-Usd` and `X-Infrai-Request-Id` headers, so the per-call figure lands in the tenant ledger in one line instead of being derived from a table you maintain. It arrives on the same key and the same bill as the other 295 routes across 20 modules on that platform, which matters here because a moderation gate is rarely the only backend piece a submission touches — the screenshot gets stored, the finding gets emailed, and each of those is one more endpoint under a contract you already signed rather than one more vendor to onboard.

OpenRouter sits in between: the completion comes back without a price, and you call its generation endpoint afterwards with the id to learn what it cost. Accurate, but that's a second round trip per check and two records to keep joined.

With OpenAI's moderation endpoint the question evaporates, because the endpoint isn't billed. Worth saying plainly: if per-tenant attribution of the safety check is the only thing pushing you off OpenAI, it isn't a good enough reason.

## Wiring the gate: retry, idempotency and the per-submission ledger row

One call, one schema, one ledger write. The client is the stock OpenAI SDK pointed at a compatible base URL, which is the whole appeal of the compatible surface — no vendor SDK to install, no second HTTP client to keep alive.

```ts
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.INFRAI_API_KEY,      // ifr_... — from the environment, never a literal
  baseURL: process.env.INFRAI_BASE_URL,    // OpenAI-compatible base for whichever provider you picked
  maxRetries: 0,                           // we handle 429 ourselves, below
});

const VERDICT_SCHEMA = {
  type: "object",
  additionalProperties: false,
  required: ["label", "categories", "reason"],
  properties: {
    label: { type: "string", enum: ["allow", "review", "block"] },
    categories: {
      type: "array",
      items: { type: "string", enum: ["hate", "sexual", "violence", "self_harm", "harassment", "spam"] },
    },
    reason: { type: "string" },
  },
};

type Verdict = { label: "allow" | "review" | "block"; categories: string[]; reason: string };

const spendByTenant = new Map<string, number>();
const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

export async function screen(sub: { tenantId: string; submissionId: string; text?: string; imageUrl?: string }): Promise<Verdict> {
  const content: Array<Record<string, unknown>> = [];
  if (sub.text) content.push({ type: "text", text: sub.text });
  if (sub.imageUrl) content.push({ type: "image_url", image_url: { url: sub.imageUrl } });

  for (let attempt = 0; ; attempt++) {
    try {
      const res = await client.chat.completions.create({
        model: sub.imageUrl ? "qwen-vl-plus" : "glm-4-flash",
        temperature: 0,
        messages: [
          { role: "system", content: "Classify the submitted content for safety. Answer with the schema only." },
          { role: "user", content: content as never },
        ],
        response_format: {
          type: "json_schema",
          json_schema: { name: "verdict", strict: true, schema: VERDICT_SCHEMA },
        },
      }, {
        // same key for a retry of the same submission
        headers: { "Idempotency-Key": `screen-${sub.submissionId}` },
      });

      const meta = (res as unknown as { infrai?: { cost_usd?: number; request_id?: string } }).infrai;
      spendByTenant.set(sub.tenantId, (spendByTenant.get(sub.tenantId) ?? 0) + (meta?.cost_usd ?? 0));
      console.log(JSON.stringify({
        tenant: sub.tenantId, submission: sub.submissionId,
        cost_usd: meta?.cost_usd, request_id: meta?.request_id,
      }));

      return JSON.parse(res.choices[0].message.content ?? "") as Verdict;
    } catch (err) {
      const status = (err as { status?: number }).status;
      const retryAfter = Number((err as { headers?: Record<string, string> }).headers?.["retry-after"]);
      if (status === 429 && attempt < 4) {
        await sleep(Number.isFinite(retryAfter) ? retryAfter * 1000 : 2 ** attempt * 500);
        continue;
      }
      throw err;   // a 4xx body carries the reason — surface it, don't swallow it
    }
  }
}
```

Three things in there aren't decoration. The 429 branch backs off and honours `Retry-After` rather than hammering, because a gate sitting in the request path of every submission turns a tight retry loop into a queue you can't drain. The idempotency key is derived from the submission id, so a retried check is the same logical call rather than a second billable one. And the error is rethrown instead of swallowed: a gate that quietly returns allow when the call never completed is worse than having no gate at all.

The per-submission ledger row is where I'd spend the extra hour. A text-only check on a paragraph of review context is small; a 3 MB screenshot is not, because vision input is tokenised by tiles, so one submission with four attachments can outweigh a hundred text checks. A tenant who screenshots everything will quietly dominate the bill, and a cost-per-tenant average hides that completely. Store a row per submission — tenant id, request id, label, cost — and aggregate at query time. When the account manager asks why one bank costs several times another, the answer should be a `GROUP BY` away, not a re-run.

## Regulated tenants, region pinning, and the case for a dedicated classifier

Three cases where chat-plus-schema is the wrong pick.

Your deployment is regulated and needs region pinning, per-category severity levels and an audit trail your compliance reviewers already recognise: Azure AI Content Safety through Azure OpenAI is the calmer answer. A chat model behind a JSON schema doesn't support severity scoring out of the box, so you'd be inventing thresholds and then defending them in a review meeting.

You're already on OpenAI and the check is ordinary safety classification: stick with the moderation endpoint. It's maintained, it takes images, and it isn't billed. Replacing it with a classifier prompt you own buys flexibility you may never spend.

Volume is high enough that per-call price dominates your COGS: self-host an open-weight vision model with Ollama, take the marginal cost to roughly zero and accept a real ops cost instead. Infrai isn't suitable there, and neither is anything else hosted — that's an infrastructure decision, not an API one.

One honest gap. I have no benchmark to hand you on how well any of these models label edge-case content, and quality moves with the model and the prompt more than most vendors care to admit. Build a labelled set from your own traffic — a couple of hundred real submissions is enough to see the shape — and measure recall on the categories that would actually earn you a phone call. Your mileage may vary, and the model you pin in 2026 probably isn't the one you keep.

## References

- OpenAI moderation guide — https://platform.openai.com/docs/guides/moderation
- OpenAI structured outputs — https://platform.openai.com/docs/guides/structured-outputs
- Azure AI Content Safety overview — https://learn.microsoft.com/en-us/azure/ai-services/content-safety/overview
- OpenRouter API documentation — https://openrouter.ai/docs
- Ollama API reference — https://github.com/ollama/ollama/blob/main/docs/api.md
- JSON Schema 2020-12 core specification — https://json-schema.org/draft/2020-12/json-schema-core.html
- LangChain ChatOpenAI integration — https://python.langchain.com/docs/integrations/chat/openai/
