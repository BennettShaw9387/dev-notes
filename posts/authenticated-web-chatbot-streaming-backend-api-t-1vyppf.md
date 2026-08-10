# Authenticated Web Chatbot Streaming: Backend API Trade-offs Beyond SDK Lock-In

Short answer: keep the model call behind your application backend, expose one authenticated streaming route, and define a provider-neutral event contract before choosing an SDK. That gives an in-app chatbot a small first-call surface, keeps credentials out of the browser, and leaves room to change inference providers without rewriting the UI.

| Decision axis | Default choice | Change it when |
| --- | --- | --- |
| Browser boundary | Same-origin `POST` with a streamed response | A separate client or public integration needs its own auth contract |
| Stream format | Server-sent events over HTTP | The client must send frequent messages while one response is open |
| Conversation state | Persist turns on the backend | The chat is disposable and has no audit or resume requirement |
| Model interface | A tiny internal adapter | You need provider-specific features that cannot fit the adapter |

The matrix is intentionally boring. Boring boundaries are easier to benchmark, inspect, and replace.

Start small.

## What should an authenticated web chatbot streaming API own?

The backend should own four things: identity, authorization, conversation state, and the upstream model request. The browser should send a user message and receive typed events. It should never receive a long-lived provider credential, and it should not decide which tenant or conversation is charged.

Authentication is not authorization. A valid session proves who is calling; a conversation lookup must still prove that this user can write to that conversation. Make that check before starting the stream. Once bytes are sent, changing an HTTP status is no longer a useful error channel.

Use one request identifier for the whole turn. Store the user message and a `pending` state before the upstream call, then mark it `complete` or `aborted` when the stream ends. A client retry can present the same identifier and receive the stored result instead of creating a second turn. This is the part SDK examples usually omit because it is application policy, not model syntax. It also gives support and billing a stable handle: when a user reports a partial answer, you can inspect one record and see the authorization decision, the selected model policy, the first-byte timestamp, and the final state without trying to reconstruct a request from browser logs. Keep the state transition monotonic; a late network callback must not turn an already-aborted turn back into `complete`.

The response contract can stay small:

```ts
type ChatEvent =
  | { type: "start"; turnId: string }
  | { type: "delta"; text: string }
  | { type: "done"; usage?: { inputTokens: number; outputTokens: number } }
  | { type: "error"; code: "rate_limited" | "upstream" | "cancelled" };

async function streamChat(input: {
  userId: string;
  conversationId: string;
  message: string;
  turnId: string;
}): Promise<AsyncIterable<ChatEvent>> {
  await assertCanWrite(input.userId, input.conversationId);
  await savePendingTurn(input);

  return providerAdapter.stream({
    conversationId: input.conversationId,
    message: input.message,
    signal: requestAbortSignal(),
  });
}
```

The adapter is the seam. Its job is translating a provider's chunks into `delta` events and translating provider errors into your finite error codes. Keep prompt assembly, retrieval, and persistence outside it. That separation makes a model swap a controlled change instead of a front-end migration.

## How do transport, auth, and retries shape the backend API?

For a first-party web app, an HTTP `POST` that returns an SSE-style stream is a pragmatic default. The request body can carry JSON, the existing session cookie can authenticate it, and the server can enforce origin and CSRF policy like any other state-changing route. A browser `EventSource` connection is less flexible because it is a `GET` and does not provide a request body; that makes long prompts, attachments, and explicit request metadata awkward.

Do not retry after the first `delta` event. At that point the user has observed part of an answer, so a second upstream call can duplicate text or spend twice for one turn. Retry only before streaming begins, with a bounded policy for a rate-limit response, and persist the turn identifier so a network retry remains idempotent.

That boundary is easy to explain and easy to test.

Cancellation is part of the contract too. Tie the browser's abort signal to the server request and then to the upstream call. A closed tab should stop generation and move the turn to `aborted`; it should not leave an expensive request running until a timeout.

Proxies can buffer streaming responses. Verify that the route forwards chunks as they arrive, disables response transformation, and sends a keep-alive comment if an intermediary has an idle timeout. These are deployment settings, not reasons to add another protocol.

## Where does an SDK help, and where does it become lock-in?

An SDK can remove repetitive serialization and give you typed helpers. It can also smuggle provider-specific concepts into every layer: model names in database rows, exception classes in UI code, and retry behavior hidden inside a convenience method. That is configuration bloat disguised as ergonomics.

Start with the wire contract. A direct `fetch` call in one adapter is often enough for a streaming route. Add an SDK when it demonstrably reduces code you own or when a capability, such as a specialized tool call, cannot be represented by your adapter. Keep the SDK import in that adapter and test the adapter against recorded, provider-neutral events.

The same rule applies to embeddings and retrieval. An embedding is a vector representation used to find relevant text; it is not a license to put a vendor's vector schema into your conversation API. Store the source chunk, model metadata, and dimensions explicitly. The embeddings guide documents the basic retrieval pattern, while the Prompt Engineering Guide is a useful reminder that prompt structure affects output independently of transport.

I'm not sure a single adapter can cover every future feature. Your mileage may vary. The useful test is whether the next provider requires changes to the browser contract. If it does, the adapter boundary is too thin or the feature is genuinely provider-specific and should be exposed as an explicit capability.

## Which alternatives are better for specific chatbot workloads?

WebSockets earn their operational cost when the conversation is genuinely bidirectional: live interruption, audio frames, or multiple simultaneous updates that must share one connection. They bring connection lifecycle work, reconnect semantics, and multi-instance routing. For ordinary request-then-stream text, that machinery is unnecessary.

Buffered JSON is the better choice for short, deterministic answers such as classification or a status lookup. It is simpler to retry and test, and it avoids teaching every client to parse partial state. Streaming only helps when perceived latency matters enough to justify a more complicated failure path.

A direct browser-to-provider call is suitable for a throwaway prototype only when its credential and abuse model are acceptable. It is a poor default for a paid, authenticated application because the browser cannot be your enforcement point for tenant limits, audit records, or conversation ownership.

The catch is that a backend relay does not solve answer quality. Retrieval quality, prompt design, and evaluation still decide whether the chatbot is useful. Stick with a direct provider SDK when a small internal tool has no multi-tenant data, no durable history, and a provider-specific feature is the product. Pick the backend contract described here when the web app, not the model, is the long-lived asset.

## A compact test plan for the decision

Test the contract at three levels. First, an authorization test must show that a valid session cannot write another user's conversation. Second, a stream test must concatenate `delta` events into the expected text and verify that `done` or `error` is always terminal. Third, a cancellation test must show that aborting the browser request stops the upstream signal and records the turn as `aborted`.

Measure time to first byte separately from total duration. The first number predicts whether the interface feels responsive; the second describes how long a completion runs. Record both with the turn identifier, plus token usage when the provider supplies it. Averages hide the slow tail, so inspect percentiles and keep the raw samples for a comparison after every transport or deployment change.

Do not put the API key, model-selection policy, or tenant budget in front-end configuration. Expose those as backend policy. The simplest backend API is the one whose rules remain visible when the first production incident arrives.

## References

- https://platform.openai.com/docs/guides/embeddings
- https://www.promptingguide.ai
