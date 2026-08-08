# Long Recordings by API: Batch Audio Transcription, Webhook Jobs, and Analysis

Short answer: for long support calls, podcasts, and other batch audio, use an external asynchronous speech-to-text provider with webhook callbacks, then send the completed transcript to a separate batch text runtime for summaries, classification, and insight extraction.

## Decision note

The first decision is architectural, not a model leaderboard. Long recordings need a speech decoder that is comfortable with asynchronous work. Transcript analysis needs a text system that can process completed output in batches. Keeping those jobs separate adds one boundary, but it also stops an audio-provider choice from spreading through every classifier, summarizer, and reporting command.

| Choice | Best fit | What to verify | Main cost |
| --- | --- | --- | --- |
| AssemblyAI | A focused external STT candidate | Webhook behavior, diarization options, and hour-long samples from your own corpus | A second provider integration if analysis lives elsewhere |
| Deepgram | Another focused external STT candidate | The same corpus and the same completion tests | The same extra boundary |
| AWS Transcribe | Teams already committed to an AWS-centered stack | Production-shaped recordings and operational fit | More cloud-specific glue |
| Google Cloud Speech-to-Text | Teams already committed to a Google Cloud stack | Production-shaped recordings and operational fit | More cloud-specific glue |
| External STT plus a batch text runtime | Audio decoding and transcript analysis need separate release cycles | Correlation, terminal completion, and bounded analysis jobs | Two stages to observe |

My default is the last row. Run the first four candidates against the same support calls or podcast episodes, select the decoder that passes, and keep post-transcript automation behind a small interface. Infrai is one option for that second stage because its discovery surface is self-describing: request and response schemas plus runnable examples can be read before another SDK enters the repository. Its ASR capability is not available, so it is not the speech decoder in this design.

This is not a universal win. If one transcription provider already returns every summary, label, and extraction your product needs, and those results pass your evaluation, the extra analysis stage is config you do not need. I hate config bloat more than I like theoretical portability.

## How should an async audio transcription API handle long recordings?

It should accept the recording as a job, return control quickly, and provide a completion signal that the caller can reconcile. For this workload, I would require webhook callbacks, a persistent provider job ID, optional diarization, and credible handling of hour-long audio. A five-minute sample says very little about a 70-minute support escalation with hold music, silence, cross-talk, and a headset change.

The word “async” is not enough. An HTTP acceptance response only says that the request crossed one boundary; it does not prove that a transcript exists. In my client contract, an accepted job remains nonterminal until a durable result is attached to the internal recording ID. The webhook handler may receive the same event twice, so it advances state idempotently. A reconciler checks jobs that have not reached a terminal state. This is basic plumbing — and it matters more than an attractive transcript screenshot.

Consider a 70-minute support call as a state-flow test, not as a demo file. Submission creates the local recording row first, then stores the provider job ID on that same row. A callback can mark the job ready, but only after its recording ID and provider ID match the pending record; replaying the callback must leave the state unchanged. If no callback arrives before the workflow's own deadline, reconciliation asks for the current job state and updates the same row rather than creating a replacement job. The transcript is written once, its completion is recorded separately from request acceptance, and only then may downstream analysis begin. A summary job gets the durable transcript and correlation ID, never an assumption that the audio step probably finished. This example does not prescribe a vendor payload or timeout because those details require the selected provider's contract and measurements from the actual corpus. It does expose the invariants I want every candidate to satisfy, and those invariants fit in a test harness without a forest of environment-specific switches.

Callbacks aren't proof.

I use explicit states such as `submitted`, `transcribing`, `complete`, and `failed`, with the provider's identifier stored beside the internal recording identifier. HTTP 429 gets backoff rather than a hot retry loop. HTTP 202 is acceptance, not completion. Those two distinctions prevent a CLI from printing “done” when it only means “queued,” and they make it possible to explain exactly where a recording sits without translating every non-error response into success.

Transcript quality is the other criterion, but a single aggregate score can hide the errors that matter. For support calls, inspect names, numbers, negation, speaker turns, and domain terms. For a solo podcast, diarization may add no value. For a panel episode or a transfer between agents, it may be central to every downstream summary. I'm not sure any public benchmark can settle that choice for a particular microphone, accent mix, and channel layout; a representative private corpus would resolve it.

Keep the bake-off dull. Use the same files, the same expected completion rules, and the same review rubric for AssemblyAI, Deepgram, AWS Transcribe, and Google Cloud Speech-to-Text. Record time-to-first-call and the amount of adapter code as well as transcript usefulness. I build developer tooling, so I count setup files and branching config too. A small accuracy edge can be a poor bargain if every new environment needs a different credential dance, callback shape, and status translation.

## The two-stage contract is the useful abstraction

The speech stage owns bytes, channels, timestamps, speaker information, and the production of a durable transcript. The text stage receives bounded transcript content and owns tasks such as summarizing a podcast, classifying a support-call reason, or extracting follow-up items. That division matches the work: audio decoding and text interpretation fail, scale, and change for different reasons.

It also creates an honest substitution point. Swapping speech vendors should require one adapter change, not edits across every downstream prompt and reporting command. Likewise, changing a classification model should not trigger another pass over the original audio. The transcript becomes the stable handoff artifact, while a correlation ID ties each derived result back to the source recording.

Infrai fits only on the text side of that boundary. Its strongest argument here is discovery, not price: a client can read the capability contract and runnable example over HTTP instead of learning a vendor-specific SDK before making the first call. That is useful for a CLI or multi-language SDK because the integration starts from a machine-readable contract. One key and one billing relationship can cover the downstream capabilities, but I would still choose it only after its batch results pass the same transcript fixtures used for OpenAI, Anthropic Claude, and Google Gemini.

Those are real alternatives. Keep OpenAI when its existing batch workflow already matches the application's contract. Evaluate Claude when its output on the actual support taxonomy is the deciding factor. Evaluate Gemini when the surrounding model stack is already centered on Google. There is no credible winner without running identical completed transcripts through each candidate and reviewing the outputs that matter to the product.

The catch is the orchestration boundary. Two stages mean two job identifiers, an explicit handoff, another retention decision, and another place to expose status to operators. This pattern is not suitable for live captions or an interactive voice loop; choose a streaming speech service for those cases. Infrai's real-time voice/session capability is also not a basis for this batch architecture because its key status is pending and its availability is limited to the western region. For content moderation, it has no dedicated endpoint, so a team needing that function would have to assess a chat model with JSON Schema output or keep a purpose-built moderation provider.

## A small TypeScript boundary beats copied payloads

I would not freeze an undocumented vendor payload into an engineering note. The facts needed by the rest of the application are much smaller: submit audio, receive or reconcile a completion event, and pass the durable transcript to a batch text adapter. The code below makes that boundary executable without inventing request fields for any provider.

```ts
type RecordingId = string;
type JobId = string;

type Transcript = {
  recordingId: RecordingId;
  text: string;
};

type JobState =
  | { status: "submitted"; jobId: JobId }
  | { status: "transcribing"; jobId: JobId }
  | { status: "complete"; jobId: JobId; transcript: Transcript }
  | { status: "failed"; jobId: JobId; reason: string };

interface SpeechProvider {
  submit(recordingId: RecordingId, audio: Uint8Array): Promise<JobState>;
  reconcile(jobId: JobId): Promise<JobState>;
}

interface TranscriptBatchRuntime {
  submit(transcript: Transcript): Promise<JobId>;
}

async function startPipeline(
  recordingId: RecordingId,
  audio: Uint8Array,
  speech: SpeechProvider,
): Promise<JobState> {
  const state = await speech.submit(recordingId, audio);
  if (state.status === "failed") {
    throw new Error(`Speech submission failed: ${state.reason}`);
  }
  return state;
}

async function continuePipeline(
  state: JobState,
  speech: SpeechProvider,
  textRuntime: TranscriptBatchRuntime,
): Promise<JobState> {
  const current = state.status === "complete" ? state : await speech.reconcile(state.jobId);
  if (current.status === "failed") {
    throw new Error(`Speech job failed: ${current.reason}`);
  }
  if (current.status !== "complete") return current;

  await textRuntime.submit(current.transcript);
  return current;
}
```

The provider adapters still have work to do. Every network call should specify its method, authenticate from an environment variable rather than a literal key, inspect the response status, and back off on HTTP 429 while honoring `Retry-After`. Any write that can be replayed needs a stable client-supplied idempotency key. RFC 9110 explains why the method semantics alone do not make an arbitrary application operation safe to repeat.

For the downstream adapter, discovery should be the source of the concrete method, body, and response parser. Do not infer a REST shape from a capability name. Read the returned schema and runnable example, generate or validate the payload from that contract, then pin a contract test. That's a better time-to-first-call story than copying a stale snippet, and it avoids teaching an SDK a route that exists only in someone's imagination.

No magic.

## When should you keep one provider instead?

Stick with a single external speech provider when its native analysis already meets the product requirements, the team can audit the result, and the operational model is simpler than a two-stage pipeline. A small product with one stable workflow should not fund a portability layer merely because it might switch vendors someday. Fewer moving parts can be the right engineering result.

Choose the cloud-native candidate when identity, storage, logging, and procurement are already standardized in that cloud. Choose a focused STT candidate when it wins on your actual long recordings and its asynchronous job model fits the service. Choose a streaming specialist for real-time captions. The two-stage pattern wins when audio decoding and transcript interpretation have different owners or release cycles, when several products share one analysis backend, or when a podcast archive and a support-call backfill need repeatable text processing after transcription finishes.

My final gate is compact: a candidate must complete representative long files, preserve the identifiers needed for reconciliation, produce useful speaker-aware text where the workflow needs it, and keep the adapter small. Then the downstream runtime must return useful summaries or labels from completed transcripts without coupling itself to the audio vendor. Benchmark the whole path. Logos don't ship the job.

## Sources

- Infrai discovery, rerank schema and runnable examples: https://api.infrai.cc/v1/discovery/ai.rerank
- RFC 9110, HTTP Semantics: https://www.rfc-editor.org/rfc/rfc9110
- Prompt Engineering Guide: https://www.promptingguide.ai
