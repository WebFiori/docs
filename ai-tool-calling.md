# Tool Calling

Tool calling (also called function calling) lets the model invoke functions you define, whether to fetch data, run calculations, or take actions, and then use the results in its reply. WebFiori AI supports both an automatic execution loop and a manual loop, on every provider.

<meta name="description" content="Define and call tools (functions) with WebFiori AI: the Tool class, auto-execute mode, the manual loop, multi-modal ToolResponse, and LazyTool.">

## Defining a Tool

A `Tool` has a name, a description, a JSON-Schema parameter definition, and a handler that receives the decoded arguments:

```php
use WebFiori\Ai\Tool\Tool;

$weatherTool = new Tool(
    'get_weather',
    'Get the current weather for a location',
    [
        'type' => 'object',
        'properties' => [
            'location' => ['type' => 'string', 'description' => 'City name'],
        ],
        'required' => ['location'],
    ],
    function (array $args): string {
        $data = [
            'location' => $args['location'] ?? 'Unknown',
            'temperature' => 22,
            'condition' => 'sunny',
        ];

        return json_encode($data);
    }
);
```

The description and schema are what the model uses to decide when and how to call the tool, so make them clear.

## Auto-Execute Mode

The simplest approach is to pass your tools and set `auto_execute_tools`. The library then runs the whole loop for you: the model requests tools, the library executes them and feeds the results back, and this repeats until the model produces a final answer.

```php
use WebFiori\Ai\ChatOption;
use WebFiori\Ai\Message;

$response = $client->chat(
    [
        new Message('system', 'You are a helpful assistant. Use tools when appropriate.'),
        new Message('user', 'What is the weather in London?'),
    ],
    [
        ChatOption::TOOLS               => [$weatherTool],
        ChatOption::AUTO_EXECUTE_TOOLS  => true,
        ChatOption::MAX_TOOL_ITERATIONS => 5,   // safety cap on the loop
    ]
);

echo $response->getMessage()->getContent();
```

## Manual Mode

For full control, omit `auto_execute_tools`. Detect tool calls, execute them yourself, append the results as `tool` messages, and call `chat()` again:

```php
use WebFiori\Ai\Message;
use WebFiori\Ai\Tool\ToolResult;

$messages = [
    new Message('system', 'Use tools when appropriate.'),
    new Message('user', 'What is the weather like in Paris?'),
];

$response = $client->chat($messages, [ChatOption::TOOLS => [$weatherTool]]);

if ($response->hasToolCalls()) {
    $messages[] = $response->getMessage();      // record the assistant's tool request

    foreach ($response->getMessage()->getToolCalls() as $call) {
        $result = $weatherTool->execute($call->getArguments());
        $messages[] = new Message('tool', '', [], new ToolResult($call->getId(), (string) $result));
    }

    $response = $client->chat($messages, [ChatOption::TOOLS => [$weatherTool]]);
}

echo $response->getMessage()->getContent();
```

Each `ToolCall` exposes `getId()`, `getName()`, and `getArguments()`; the matching `ToolResult` ties the output back to the call via the tool-call ID.

## Tools That Return Images

A tool can return a `ToolResponse` carrying both text and image parts, so the model can *see* generated visuals (e.g. a chart) and describe them:

```php
use WebFiori\Ai\ContentPart;
use WebFiori\Ai\Tool\Tool;
use WebFiori\Ai\Tool\ToolResponse;

$chartTool = new Tool(
    'generate_chart',
    'Generates a chart image',
    ['type' => 'object', 'properties' => ['title' => ['type' => 'string']]],
    function (array $args): ToolResponse {
        $png = generateChartPng($args['title']); // your image generation

        return ToolResponse::withImages(
            json_encode(['title' => $args['title'], 'status' => 'generated']),
            [ContentPart::imageBase64(base64_encode($png), 'image/png')]
        );
    }
);

$response = $client->chat(
    [new Message('user', 'Generate a Q3 revenue chart and describe it.')],
    [ChatOption::TOOLS => [$chartTool], ChatOption::AUTO_EXECUTE_TOOLS => true]
);
```

## Deferred Construction with LazyTool

If a tool is expensive to set up (e.g. a database connection), use `LazyTool`. Its factory runs only when the tool is actually called:

```php
use WebFiori\Ai\Tool\LazyTool;

$dbTool = new LazyTool(
    'search_database',
    'Search the product database',
    ['type' => 'object', 'properties' => ['query' => ['type' => 'string']], 'required' => ['query']],
    function () {
        $db = connect_to_database();               // runs on first call only
        return function (array $args) use ($db): string {
            return json_encode($db->search($args['query']));
        };
    }
);
```

## Where to Next

See [Streaming](learn/ai-streaming) to stream responses token by token, or [RAG](learn/ai-rag) to ground responses in your own documents.
