# ADR-0042: AI: Response Recording & Replay — VCR-Style HTTP Fixtures

**Date:** 2026-08-23
**Status:** Accepted

## Context

Integration tests that hit real AI APIs are slow, expensive, non-deterministic,
and require API keys in CI. Developers building applications with this library
need a way to test their AI integration code reliably without live API calls.

`FakeHttpClient` already exists for unit tests where the developer hand-writes
response payloads. The missing piece is **VCR-style recording**: make real API
calls once, save the responses, replay them in future test runs.

## Decision

Add two `HttpClientInterface` decorator classes:

- `RecordingHttpClient` — wraps any HTTP client, records responses to JSON fixture files
- `ReplayHttpClient` — loads fixtures from disk, replays them without network calls

### Design Decisions

**1. Fixture level: HTTP (not AI)**

Fixtures store raw HTTP request/response pairs, not `ChatResponse` objects.
This makes fixtures provider-agnostic and works automatically for all methods
(chat, embed, image) without needing serialization logic per response type.

```json
{
  "name": "ask_what_is_php",
  "recorded_at": "2026-08-23T14:00:00Z",
  "fingerprint": {
    "url": "https://api.openai.com/v1/chat/completions",
    "messages_hash": "sha256:abc123..."
  },
  "streaming": false,
  "response": {
    "status": 200,
    "headers": {"Content-Type": "application/json"},
    "body": { ... }
  }
}
```

**2. Matching: URL + messages hash (configurable)**

Default matching uses `URL + hash of the messages array` only. Generation
parameters (`temperature`, `max_tokens`, `top_p`, etc.) are excluded from the
fingerprint because they don't change what conversation is being tested.

A custom `FingerprintStrategy` can be provided when stricter matching is needed
(e.g., including tools or JSON schema in the fingerprint).

```php
$replayer = new ReplayHttpClient($path);  // default: URL + messages hash
$replayer = new ReplayHttpClient($path, new FullBodyFingerprintStrategy()); // strict
```

**3. Streaming: raw chunks stored separately**

Streaming fixtures store the actual SSE chunks received over the wire, not a
reconstructed response. This ensures the real SSE parsing code path is exercised
during replay.

```json
{
  "streaming": true,
  "chunks": [
    "data: {\"choices\":[{\"delta\":{\"content\":\"Hello\"}}]}\n\n",
    "data: [DONE]\n\n"
  ]
}
```

Non-streaming fixtures store the full response body as a JSON object.

**4. Miss policy: always throw**

When `ReplayHttpClient` cannot find a matching fixture, it always throws with a
descriptive error. No silent fallback to live HTTP — that would defeat the
purpose and reintroduce CI dependencies on API keys.

```
FixtureNotFoundException: No fixture matched:
  URL: POST https://api.openai.com/v1/chat/completions
  Messages hash: sha256:abc123...
  Searched: /path/to/fixtures (5 files)
  Hint: Run with RecordingHttpClient to record this response.
```

**5. Fixture naming: content-based matching, human-readable filenames**

Filenames are decorative. `RecordingHttpClient` auto-generates names as
`{provider}_{endpoint}_{short_hash}.json`, but developers can rename files
to anything meaningful (e.g., `test_weather_tool.json`). Matching is always
done by the fingerprint stored **inside** the fixture, never by filename.

This allows developers to:
- Record automatically, then rename files to meaningful names
- Write fixtures by hand with descriptive names
- Reorganise fixtures into subdirectories freely

### Usage

```php
// Recording (run once against real API)
$recorder = new RecordingHttpClient(
    inner: new CurlHttpClient(),
    path: __DIR__ . '/fixtures',
);
$provider->setHttpClient($recorder);
$provider->chat($messages); // Saves fixture to disk

// Replay (run in tests, no API key needed)
$replayer = new ReplayHttpClient(__DIR__ . '/fixtures');
$provider->setHttpClient($replayer);
$response = $provider->chat($messages); // Loaded from fixture
```

### API Key Scrubbing

Recorded fixtures automatically redact `Authorization`, `x-api-key`, and
`x-goog-api-key` headers. Request and response bodies are never modified
(API keys do not appear in bodies by design).

## Alternatives Considered

**AI-level fixtures (`ChatResponse` objects):**
Would be more readable but requires serialization per response type and breaks
when providers change their response format. HTTP-level is more durable.

**Full body hash matching:**
Too brittle — any change to temperature or tools creates a new hash even if
the conversation is identical. URL + messages hash is the right semantic unit.

**Fall-through on miss:**
Allows PASSTHROUGH (real HTTP) or RECORD-on-miss modes. Rejected because it
reintroduces CI dependencies on API keys. Loud failure is the correct default.

**Filename-based matching:**
Ties fixture identity to filename — renaming breaks tests. Content-based
matching (fingerprint inside the file) is more robust.

## Consequences

**Easier:**
- Integration tests run in CI without API keys
- Tests are deterministic and fast
- Developers can record once and commit fixtures alongside code
- All providers work automatically (no per-provider fixture logic)

**Harder:**
- Fixtures become stale when provider response formats change
- Recording must be re-run when conversations change significantly
- Streaming fixtures are larger (raw chunks) than a synthesized equivalent
