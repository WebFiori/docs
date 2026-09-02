# Streaming

Streaming delivers the model's response token-by-token as it is generated, instead of waiting for the full reply. This is ideal for chat UIs and long responses. WebFiori AI streams through `streamChat()` with callback functions.

<meta name="description" content="Stream AI responses token-by-token with WebFiori AI using streamChat and onToken/onComplete/onError callbacks, including Server-Sent Events.">

## Basic Streaming

`streamChat()` takes the messages plus an `onToken` callback that fires for each chunk. Optional `onComplete` and `onError` callbacks handle the end of the stream and failures:

```php
use WebFiori\Ai\Provider\OpenAI\OpenAIClient;
use WebFiori\Ai\Provider\OpenAI\OpenAIClientConfig;
use WebFiori\Ai\Message;

$client = new OpenAIClient(new OpenAIClientConfig(apiKey: 'sk-...', model: 'gpt-4o'));

$client->streamChat(
    messages: [
        new Message('user', 'Write a short poem about PHP.'),
    ],
    onToken: function (string $token): void {
        echo $token;   // print each chunk as it arrives
        flush();
    },
    onComplete: function ($response): void {
        echo PHP_EOL.'--- done ---'.PHP_EOL;
        echo 'Finish reason: '.$response->getFinishReason().PHP_EOL;
    },
    onError: function (\Throwable $e): void {
        echo PHP_EOL.'Error: '.$e->getMessage().PHP_EOL;
    },
);
```

The method signature is:

```php
public function streamChat(
    array $messages,
    callable $onToken,                 // function(string $token): void
    ?callable $onComplete = null,      // function(ChatResponse $response): void
    ?callable $onError = null,         // function(Throwable $e): void
    array $options = []
): void;
```

`streamChat()` works across all providers and accepts the same options array as `chat()` (`model`, `temperature`, etc.).

## The onComplete Callback

`onComplete` receives the assembled `ChatResponse`, the same object `chat()` would return, so you can read the full text, finish reason, and usage after streaming ends:

```php
onComplete: function ($response): void {
    $full = $response->getMessage()->getContent();
    $usage = $response->getUsage();
    // persist $full, log $usage, etc.
},
```

## Streaming Over Server-Sent Events

For a browser client, emit each token as a Server-Sent Event. Set the SSE headers, then write inside `onToken`:

```php
header('Content-Type: text/event-stream');
header('Cache-Control: no-cache');
header('X-Accel-Buffering: no');

$client->streamChat(
    messages: [new Message('user', $_GET['q'] ?? 'Hello')],
    onToken: function (string $token): void {
        echo 'data: '.json_encode(['token' => $token])."\n\n";
        ob_flush();
        flush();
    },
    onComplete: function ($response): void {
        echo 'event: done'."\n";
        echo 'data: '.json_encode(['content' => $response->getMessage()->getContent()])."\n\n";
        ob_flush();
        flush();
    },
);
```

## Error Handling

Provide `onError` to handle mid-stream failures without an uncaught exception. Transport-level failures before the stream starts propagate as exceptions from `streamChat()` itself, so wrap the call in `try/catch` if you need to handle both:

```php
use WebFiori\Ai\Exception\AiException;

try {
    $client->streamChat($messages, $onToken, $onComplete, function (\Throwable $e): void {
        // stream-time error surfaced by the provider handler
    });
} catch (AiException $e) {
    // failure before/while establishing the stream
}
```

## Where to Next

See [Tool Calling](learn/ai-tool-calling) to let the model call your functions, or [Basic Chat](learn/ai-basic-chat) for the non-streaming equivalent.
