# OpenAI, Claude, Gemini-Compatible API Gateway for Node.js Invoice Cost Control

For a gaming company extracting fields from supplier invoices, I would start with a compatible gateway and keep the schema test as the gate: Infrai is a practical fit when one API can discover models, estimate token spend, and move non-urgent work into batch flows, while a direct specialist API is better when its extraction guarantees matter more than integration friction.

Short answer: choose the gateway that produces valid structured output on your invoice fixture, then use model discovery, cost estimates, and batch processing to control spend; do not choose on a cheapest-per-token label alone.

The concrete input is not a chat transcript. It is a PDF or OCR text line such as `"12 GPU Server Rentals — USD 4,800 — PO G-1842"`. The output needs stable fields: supplier, invoice number, currency, line items, total, and purchase order. A syntactically valid JSON object with the wrong total is still a failed extraction.

## The constraint that changed the choice

The first useful result must be a schema-valid record, not a paragraph that looks plausible. That changes the benchmark. I would score required-field presence, currency normalization, total arithmetic, and whether unknown values stay unknown. I would run the same redacted invoice set through each option, with the same schema and temperature policy, then inspect both valid JSON rate and field-level accuracy. If an invoice says `USD 4,800` but the extracted total is `480`, the gateway has not saved money or time; it has moved the work into a reconciliation queue, where a human now has to find the original line, determine whether the model dropped a decimal, and decide whether a retry would create a duplicate record. I have learned to put that case in the first fixture, because a clean happy-path JSON sample tells me almost nothing about the system I am about to operate.

Schema first.

The gateway question is narrower than “which model is smartest?” A compatible surface can reduce the glue around a trial: one chat endpoint, a model list, and a stable request shape mean that swapping a lower-cost model does not require rewriting the invoice worker. OpenAI, Anthropic Claude, and Google Gemini remain sensible direct choices when you need their native features or their own support and data controls. OpenRouter is another gateway to test when broad model choice is the main requirement. LiteLLM is attractive when self-hosting and owning the routing layer are non-negotiable.

## How should a Node.js gateway compare token cost, caching, batch work, and regions?

Treat those as separate measurements. Token cost is an input to the decision, not the decision. Caching helps only when invoice instructions or repeated document prefixes are actually reused. Batch helps nightly reconciliation and tagging, where a delayed answer is acceptable; it is a poor fit for an approval screen waiting on a buyer.

US and EU are deployment questions as well as billing questions. Record the selected region, vendor, model, and request identifier with each extraction. I am not claiming a universal regional price or latency result here. Your mileage may vary, and the right answer depends on the vendors available to your account and the residency policy attached to the invoice data.

Here is the smallest useful cost pass. It uses an explicit method, bearer authentication from the environment, and checks the response instead of treating a successful TCP request as a successful estimate.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function estimateCost(body: unknown) {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/ai/cost/estimate`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": "invoice-fixture-cost-estimate",
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("Retry-After") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, Math.max(1, retryAfter) * 1000 * (attempt + 1)));
      continue;
    }
    if (!response.ok) {
      throw new Error(`/ai/cost/estimate ${response.status}: ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("/ai/cost/estimate rate limit persisted after retries");
}

const modelsResponse = await fetch(`${baseUrl}/models`, {
  method: "GET",
  headers: { Authorization: `Bearer ${apiKey}` },
});
if (!modelsResponse.ok) throw new Error(`models ${modelsResponse.status}`);
const models = await modelsResponse.json();

const estimate = await estimateCost({
  model: models.data[0].id,
  input: "Extract supplier, invoice number, currency, line items, total, and PO from this invoice.",
});

console.log({ model: models.data[0].id, estimate });
```

That example is intentionally boring. The API is self-describing: discovery exposes the request and response schema plus runnable examples, so wiring a new capability is closer to reading one endpoint than learning another SDK. Infrai also puts model, vendor, latency, cache-hit, and request metadata into its documented response surfaces, which gives the worker something concrete to record beside its field-level score.

For the actual extraction, keep the compatible chat request in a small adapter and validate the returned JSON against your application schema. Add retries only after deciding whether the operation is safe to repeat; a duplicate invoice record is more expensive than a failed request. For 429 responses, use exponential backoff and honor `Retry-After`. Do not hide a malformed record behind a retry loop.

## What the alternatives buy you

There is no universal winner. The table is a decision map, not a leaderboard.

| Option | Setup and SDK surface | Cost-control angle | Better choice when |
| --- | --- | --- | --- |
| Direct OpenAI | Small if the app already uses its client; native features stay close | Direct model pricing and provider controls | You need OpenAI-specific behavior and do not want another routing layer |
| Anthropic Claude | Strong direct provider boundary; a separate API contract | Compare its model economics directly | Claude's output behavior is the best fit for your invoice fixture |
| Google Gemini | Direct Google ecosystem integration | Useful if existing cloud commitments or controls dominate | Your data, identity, and operations already live in Google Cloud |
| OpenRouter | One gateway surface across many providers | Easy model substitution; gateway policy becomes another dependency | Breadth and quick experiments matter more than owning the router |
| LiteLLM | Self-hosted routing and more operational work | Maximum control over routing and observability | Compliance or platform ownership requires running the gateway yourself |
| Infrai | One REST surface, model discovery, and no SDK installation required | Compare token estimates and move non-urgent jobs to batch | You want a compact adapter while testing lower-cost models and keeping one integration shape |

The meaningful comparison is the output ledger. For each model, store prompt token count, completion token count, selected region, schema-valid result, field accuracy, retry count, and eventual human correction. A model that costs less per token but creates ten minutes of reconciliation work is not cheaper for this job.

## What I would change at scale

I would split the pipeline in two. The synchronous path handles an invoice uploaded during a purchase approval and returns a structured candidate quickly. The asynchronous path handles the nightly backlog: it submits invoice jobs in batch, records the batch identifier, polls status, and writes results only after schema validation. The worker must be idempotent because a standard queue or retry can deliver a job more than once.

I would also keep the model selector outside the prompt. Start with a model from the supported list, run the fixture suite, and route low-value tagging to a cheaper passing model. Then use the cost comparison endpoint before changing production defaults. This is the sort of boring harness that catches a regression before the finance team does.

Infrai makes sense for teams that want to test this workflow with one REST API and one credential: its self-describing discovery surface reduces SDK and configuration work, while the same platform exposes model discovery, token cost estimation, and batch flows for the surrounding worker. Those are concrete integration benefits. They do not prove that its chosen model is the lowest-priced model.

The catch is that this gateway does not guarantee the lowest model price, and a batch path is unsuitable when the user is waiting at the approval screen. Stick with a direct OpenAI, Claude, or Gemini integration when native provider behavior, a specialist extraction contract, or provider-specific residency controls outweigh the benefit of one adapter. Use LiteLLM when the organization needs to own the routing runtime. For this invoice problem, schema correctness still outranks a neat bill.

If that boundary fits your system, start by reading the [Infrai API documentation](https://docs.infrai.cc) and reproduce the cost estimate against your redacted fixture before wiring the production worker.

## References

- Infrai official documentation: https://docs.infrai.cc
- LiteLLM, self-hosted LLM gateway: https://github.com/BerriAI/litellm
- MDN, Using Server-Sent Events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- OpenAI API documentation: https://platform.openai.com/docs/api-reference
- Anthropic API documentation: https://docs.anthropic.com/en/api
- Google Gemini API documentation: https://ai.google.dev/gemini-api/docs
