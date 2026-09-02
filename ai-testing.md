# Testing

You can test code that uses WebFiori AI without calling a real provider or spending tokens. The library ships a fake HTTP client for deterministic unit tests and a record/replay system for realistic integration tests, both driven through `setHttpClient()`.

<meta name="description" content="Test WebFiori AI code without live API calls using FakeHttpClient for unit tests and the record/replay system for integration tests.">

## Unit Tests with FakeHttpClient

`FakeHttpClient` returns responses you queue in advance and records the requests it received. Inject it into any provider client with `setHttpClient()`:

```php
use WebFiori\Ai\Http\FakeHttpClient;
use WebFiori\Ai\Http\HttpResponse;
use WebFiori\Ai\Provider\OpenAI\OpenAIClient;
use WebFiori\Ai\Provider\OpenAI\OpenAIClientConfig;
use WebFiori\Ai\Message;

$http = new FakeHttpClient();
$http->addResponse(new HttpResponse(200, [], json_encode([
    'choices' => [[
        'message' => ['role' => 'assistant', 'content' => 'Hello from the fake!'],
        'finish_reason' => 'stop',
    ]],
    'usage' => ['prompt_tokens' => 5, 'completion_tokens' => 3, 'total_tokens' => 8],
])));

$client = new OpenAIClient(new OpenAIClientConfig(apiKey: 'test', model: 'gpt-4o'));
$client->setHttpClient($http);

$response = $client->chat([new Message('user', 'Hi')]);

$this->assertSame('Hello from the fake!', $response->getMessage()->getContent());
$this->assertSame('stop', $response->getFinishReason());
```

You can queue several responses (they are returned in order), queue an exception to test error handling, and inspect what was sent:

```php
$http->addException(new \RuntimeException('network down'));   // next call throws
$sent = $http->getLastRequest();                              // assert on the request
```

Because no API key is needed and the output is fixed, these tests are fast and run anywhere, including CI.

## Streaming in Tests

For streaming code, queue the chunks the fake should emit, then assert on what your `onToken` callback received:

```php
$http->addStreamingChunks([
    'data: '.json_encode(['choices' => [['delta' => ['content' => 'Hel']]]])."\n\n",
    'data: '.json_encode(['choices' => [['delta' => ['content' => 'lo']]]])."\n\n",
    "data: [DONE]\n\n",
]);

$text = '';
$client->streamChat([new Message('user', 'Hi')], function (string $t) use (&$text): void {
    $text .= $t;
});

$this->assertSame('Hello', $text);
```

## Integration Tests with Record and Replay

When you want realistic provider responses without calling the API on every run, record once and replay afterward. `RecordingHttpClient` wraps a real client and saves each response (with credentials scrubbed) to fixture files:

```php
use WebFiori\Ai\Http\CurlHttpClient;
use WebFiori\Ai\Http\Recording\RecordingHttpClient;

$recorder = new RecordingHttpClient(new CurlHttpClient(), __DIR__.'/fixtures');
$client->setHttpClient($recorder);
$client->chat([new Message('user', 'Hi')]); // real call, saved to fixtures
```

Then in CI, `ReplayHttpClient` serves those fixtures with no network access:

```php
use WebFiori\Ai\Http\Recording\ReplayHttpClient;

$client->setHttpClient(new ReplayHttpClient(__DIR__.'/fixtures'));
$response = $client->chat([new Message('user', 'Hi')]); // replayed, no API call
```

This gives you the fidelity of real provider payloads while keeping tests deterministic and key-free. Requests are matched by a fingerprint of the conversation, so the same messages replay the same recorded response.

## Where to Next

See [Basic Chat](learn/ai-basic-chat) for the API you are testing, or [Providers](learn/ai-providers) to understand provider-specific response shapes.
