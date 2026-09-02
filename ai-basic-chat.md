# Basic Chat

The chat completion is the core operation of WebFiori AI: you send an array of `Message` objects and receive a `ChatResponse`. This page covers building messages, reading the response, and the common per-call options.

<meta name="description" content="Send chat completions with WebFiori AI: build messages, read the ChatResponse, and use per-call options like temperature and max tokens.">

## Sending Messages

A conversation is an array of `Message` objects, each with a role and content:

```php
use WebFiori\Ai\Provider\OpenAI\OpenAIClient;
use WebFiori\Ai\Provider\OpenAI\OpenAIClientConfig;
use WebFiori\Ai\Message;

$client = new OpenAIClient(new OpenAIClientConfig(apiKey: 'sk-...', model: 'gpt-4o'));

$response = $client->chat([
    new Message('system', 'You are a concise assistant.'),
    new Message('user', 'Explain what a closure is in PHP.'),
]);

echo $response->getMessage()->getContent();
```

The roles are `system` (instructions), `user` (input), `assistant` (prior model replies), and `tool` (tool results). For readability you can also use the factory helpers:

```php
Message::system('You are a concise assistant.');
Message::user('Explain what a closure is in PHP.');
Message::assistant('A closure is an anonymous function that can capture variables...');
```

## Reading the Response

`chat()` returns a `ChatResponse`:

```php
$response = $client->chat([new Message('user', 'Hello!')]);

$message = $response->getMessage();        // the assistant Message
echo $message->getContent();               // the reply text
echo $message->getRole();                  // 'assistant'

echo $response->getModel();                // model that produced the reply
echo $response->getFinishReason();         // e.g. 'stop', 'length', 'tool_calls'
echo $response->getRequestId();            // provider/request identifier

$usage = $response->getUsage();            // token accounting (may be null)
if ($usage !== null) {
    echo $usage->getPromptTokens();
    echo $usage->getCompletionTokens();
    echo $usage->getTotalTokens();
}
```

## Multi-Turn Conversations

To continue a conversation, append the assistant's reply and the next user message, then send the full history again:

```php
$history = [
    new Message('system', 'You are a helpful tutor.'),
    new Message('user', 'What is recursion?'),
];

$response = $history[] = $client->chat($history)->getMessage();
$history[] = new Message('user', 'Give me a PHP example.');

$response = $client->chat($history);
echo $response->getMessage()->getContent();
```

For managed history with pluggable storage, see the conversation helpers in the library's examples.

## Common Options

Pass a second array argument to control the request. Use the `ChatOption` constants to avoid typos:

```php
use WebFiori\Ai\ChatOption;

$response = $client->chat($messages, [
    ChatOption::MODEL       => 'gpt-4o-mini', // override the default model
    ChatOption::TEMPERATURE => 0.2,           // 0.0 = deterministic, higher = more random
    ChatOption::MAX_TOKENS  => 512,           // cap the response length
    ChatOption::TOP_P       => 0.9,
]);
```

The same options work across providers. Provider-specific behavior (such as which models exist) still depends on the provider you configured.

## Error Handling

Chat calls throw exceptions that all extend `AiException`, so you can catch broadly or specifically:

```php
use WebFiori\Ai\Exception\AiException;
use WebFiori\Ai\Exception\AuthenticationException;
use WebFiori\Ai\Exception\RateLimitException;

try {
    $response = $client->chat([new Message('user', 'Hello')]);
} catch (AuthenticationException $e) {
    // invalid API key / credentials
} catch (RateLimitException $e) {
    // provider rate limit hit; $e->getRetryAfterSeconds() may be available
} catch (AiException $e) {
    // any other provider/transport error
}
```

## Where to Next

See [Providers](learn/ai-providers) to choose a provider and review the feature matrix, then [Configuration](learn/ai-configuration) to set up the client for your chosen provider.
