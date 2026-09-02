# Managing Conversations

Chat calls are stateless: you send the full message history every time. The `Conversation` helper manages that history for you, keeping track of turns, capping the length, and persisting the exchange through a storage backend.

<meta name="description" content="Manage multi-turn chat history with the WebFiori AI Conversation helper, including system messages, history limits, and swappable storage.">

## Basic Usage

Create a `Conversation` with a provider, a storage backend, and a session ID. Then call `send()` for each user turn; the helper appends the reply and remembers the context:

```php
use WebFiori\Ai\Conversation\Conversation;
use WebFiori\Ai\Conversation\InMemoryStorage;
use WebFiori\Ai\Provider\OpenAI\OpenAIClient;
use WebFiori\Ai\Provider\OpenAI\OpenAIClientConfig;

$provider = new OpenAIClient(new OpenAIClientConfig(apiKey: 'sk-...', model: 'gpt-4o'));

$conversation = new Conversation($provider, new InMemoryStorage(), 'session-123');
$conversation->setSystemMessage('You are a helpful assistant. Keep responses concise.');
$conversation->setMaxHistory(20);

$response = $conversation->send('What is dependency injection?');
echo $response->getMessage()->getContent();

$response = $conversation->send('Show me a PHP example.'); // remembers the earlier turn
echo $response->getMessage()->getContent();
```

`send()` returns the same `ChatResponse` that `chat()` returns, so you read the reply the same way.

## System Message and History Limit

`setSystemMessage()` sets the instruction that stays at the front of every request. `setMaxHistory()` caps how many turns are kept, which keeps requests within the model's context window on long chats:

```php
$conversation->setSystemMessage('You answer only in formal English.');
$conversation->setMaxHistory(10);
```

You can inspect the accumulated turns at any time:

```php
foreach ($conversation->getHistory() as $message) {
    echo $message->getRole().': '.$message->getContent().PHP_EOL;
}
```

## Storage Backends

The storage backend decides where history lives between requests. `InMemoryStorage` keeps it for the current process, which suits a CLI session or tests. For a web app where each request is separate, implement `ConversationStorageInterface` over your own store (database, cache, session) and pass it in:

```php
use WebFiori\Ai\Conversation\ConversationStorageInterface;

final class DatabaseConversationStorage implements ConversationStorageInterface {
    // save(), load(), exists(), delete(), listConversations()
}

$conversation = new Conversation($provider, new DatabaseConversationStorage(), $sessionId);
```

Because the storage is swappable, the same conversation code works whether history is held in memory, in a database, or in a cache.

## Where to Next

See [Basic Chat](learn/ai-basic-chat) for the underlying stateless API that `Conversation` builds on.
