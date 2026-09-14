# ADR-0045: AI: AgentTool — Delegate Tasks to Specialist AI Providers

**Date:** 2026-08-25
**Status:** Accepted

> **Amended 2026-09-14:** `AgentProfile::output_format` now accepts `string|array`
> (parity with `context`); `examples[].output` may be authored as an array of
> strings (normalized to a newline-joined string); and the inheritance strategy
> vocabulary was simplified to `merge` / `replace` (`concat` retained as a
> deprecated alias). See Design Decision 8 for the inheritance model.

## Context

Complex tasks benefit from specialized agents. An orchestrator model should be
able to delegate subtasks to specialist AI providers, each with their own
identity, skills, constraints, and tools. For example, a general-purpose
assistant might delegate code review to an agent backed by a code-specialized
model, or delegate translation to an agent fine-tuned for multilingual tasks.

This is the "tool as agent" pattern — the orchestrator treats each specialist
agent as a tool it can invoke during its reasoning loop. The specialist runs
independently with its own system prompt, provider, and sub-tools, then returns
a result to the orchestrator like any other tool output.

The library already has a `ToolInterface` and an `auto_execute_tools` loop in
`AbstractClient`. An agent tool should compose with this existing infrastructure
rather than requiring new provider-level abstractions.

## Decision

Add `AgentTool`, `AgentProfile`, and `AgentMessageStrategy` to
`WebFiori\Ai\Tool`. The agent tool participates in the existing tool loop —
the orchestrator model calls it like any tool, and it delegates internally to
a sub-provider.

### Design Decisions

**1. AgentTool implements ToolInterface — works with existing tool loop**

AgentTool implements `ToolInterface`, so it plugs into the existing tool
execution pipeline without changes to provider formatters, response parsers,
or `AbstractClient`. The orchestrator model sees it as a regular tool with a
`task` parameter.

```php
use WebFiori\Ai\Provider\OpenAI\OpenAIClient;
use WebFiori\Ai\Provider\Google\GoogleClient;
use WebFiori\Ai\Tool\AgentTool;
use WebFiori\Ai\Tool\AgentProfile;
use WebFiori\Ai\Message;

$codeReviewer = new AgentTool(
    name: 'code_reviewer',
    description: 'Reviews code for bugs, style issues, and security vulnerabilities.',
    provider: new OpenAIClient(['api_key' => '...', 'model' => 'gpt-4o']),
    profile: new AgentProfile(
        identity: 'You are a senior code reviewer specializing in PHP.',
        skills: ['Bug detection', 'Security analysis', 'PSR-12 compliance'],
        constraints: ['Only comment on issues, not style preferences'],
    ),
    memory: null,             // optional AgentMemory
    rememberStrategy: null,   // optional RememberStrategyInterface
);

// The orchestrator uses it like any other tool
$orchestrator = new GoogleClient(['api_key' => '...', 'model' => 'gemini-2.5-pro']);
$response = $orchestrator->chat(
    [new Message('user', 'Review this code: function add($a, $b) { return $a + $b; }')],
    ['tools' => [$codeReviewer], 'auto_execute_tools' => true]
);
```

**2. AgentProfile for structured onboarding**

`AgentProfile` separates agent configuration into discrete fields: identity,
skills, instructions, constraints, output format, context, and examples. It
renders itself into a structured system prompt. This makes agent behavior
inspectable, testable, and configurable.

Content fields that render as free-form guidance accept flexible shapes:
`context` and `output_format` may each be a plain string **or** an array of
strings (arrays render as a bulleted list, one item per line), while `identity`
is always a single string. List fields (`skills`, `instructions`,
`constraints`) are always arrays.

For few-shot `examples`, each entry's `output` may be authored as a string or an
array of strings purely for readability; an array is normalized to a
newline-joined string at construction, so the stored/rendered shape stays a
single verbatim `Assistant:` response (no bullets are injected). This keeps the
few-shot signal intact while making multi-line outputs easier to author in JSON.

```php
use WebFiori\Ai\Tool\AgentProfile;

$profile = new AgentProfile(
    identity: 'You are a SQL query optimizer for PostgreSQL.',
    skills: [
        'Query plan analysis',
        'Index recommendation',
        'JOIN optimization',
    ],
    instructions: [
        'Always explain WHY a change improves performance',
        'Provide before/after EXPLAIN output when possible',
    ],
    constraints: [
        'Do not suggest schema changes unless explicitly asked',
        'Assume PostgreSQL 15+ syntax',
    ],
    outputFormat: 'Return a markdown document with sections: Analysis, Recommendations, Optimized Query.',
    context: 'The database has tables: users (10M rows), orders (50M rows), products (100K rows).',
    examples: [
        [
            'input' => 'Optimize: SELECT * FROM users WHERE email LIKE "%@gmail.com"',
            'output' => '## Analysis\nLeading wildcard prevents index usage...',
        ],
    ],
);

// Renders to a structured system prompt
echo $profile->render();
```

**3. AgentMessageStrategy enum — TASK_ONLY vs FULL_HISTORY**

The `AgentMessageStrategy` enum controls how much context the sub-agent
receives. `TASK_ONLY` (default) sends only the delegated task — stateless and
token-efficient. `FULL_HISTORY` injects the parent conversation via a setter,
giving the agent full context awareness.

```php
use WebFiori\Ai\Tool\AgentTool;
use WebFiori\Ai\Tool\AgentMessageStrategy;

// Stateless — only receives the task (default)
$translator = new AgentTool(
    name: 'translator',
    description: 'Translates text to Arabic.',
    provider: $googleClient,
    profile: new AgentProfile(identity: 'You translate text to Arabic.'),
    messageStrategy: AgentMessageStrategy::TASK_ONLY,
);

// Context-aware — receives full conversation history
$summarizer = new AgentTool(
    name: 'conversation_summarizer',
    description: 'Summarizes the conversation so far.',
    provider: $openAIClient,
    profile: new AgentProfile(identity: 'You summarize conversations concisely.'),
    messageStrategy: AgentMessageStrategy::FULL_HISTORY,
);
```

**4. Agents can have their own tools — nested architectures**

An agent's profile can include tools, enabling orchestrator → agent → sub-tools
architectures. When `AgentTool::execute()` is called, it merges profile tools
into the options and runs its own `auto_execute_tools` loop internally.

```php
use WebFiori\Ai\Tool\Tool;
use WebFiori\Ai\Tool\AgentTool;
use WebFiori\Ai\Tool\AgentProfile;

// Sub-tool available to the agent
$runQuery = new Tool(
    'run_query',
    'Executes a read-only SQL query and returns results.',
    ['type' => 'object', 'properties' => ['sql' => ['type' => 'string']], 'required' => ['sql']],
    function (array $args): string {
        // Execute query safely...
        return json_encode($results);
    }
);

// Agent with its own tools
$dbAnalyst = new AgentTool(
    name: 'db_analyst',
    description: 'Analyzes database performance and runs diagnostic queries.',
    provider: $openAIClient,
    profile: new AgentProfile(
        identity: 'You are a database performance analyst.',
        skills: ['Query optimization', 'Index analysis'],
        tools: [$runQuery], // Agent can use this tool internally
    ),
);

// Orchestrator delegates to db_analyst, which internally uses run_query
$response = $orchestrator->chat($messages, [
    'tools' => [$dbAnalyst],
    'auto_execute_tools' => true,
]);
```

**5. Profile loading — fromFile(), fromUrl(), fromArray(), fromString() factories**

Profiles can be loaded from JSON files, URLs, arrays, or plain strings. This
enables declarative agent configuration that can be changed without code
deployments. Tool references in JSON are resolved at bootstrap via
`resolveTools()`.

```php
use WebFiori\Ai\Tool\AgentProfile;

// From a JSON file
$profile = AgentProfile::fromFile('/etc/agents/code-reviewer.json');

// From a URL (e.g., config service)
$profile = AgentProfile::fromUrl('https://config.example.com/agents/reviewer.json');

// From an array (e.g., database or YAML parsed to array)
$profile = AgentProfile::fromArray([
    'identity' => 'You are a code reviewer.',
    'skills' => ['Bug detection', 'Security review'],
    'tools' => ['run_linter', 'check_types'], // tool name references
]);

// Resolve tool references against a registry
$profile->resolveTools([
    'run_linter' => $linterTool,
    'check_types' => $typeCheckerTool,
]);

// From a plain string (minimal profile)
$profile = AgentProfile::fromString('You are a helpful translator.');
```

Example JSON profile (`/etc/agents/code-reviewer.json`):

```json
{
    "identity": "You are a senior PHP code reviewer.",
    "skills": ["Bug detection", "PSR-12 compliance", "Security analysis"],
    "instructions": ["Focus on correctness over style", "Flag potential SQL injection"],
    "constraints": ["Do not rewrite code, only comment on issues"],
    "output_format": "Markdown with ## Issues and ## Suggestions sections",
    "examples": [
        {
            "input": "Review: $name = $_GET['name']; echo $name;",
            "output": "## Issues\n- XSS vulnerability: user input echoed without escaping"
        }
    ],
    "tools": ["run_linter", "check_types"],
    "metadata": {"version": "1.2", "author": "team-platform"}
}
```

**6. Context injection via setter (not argument)**

`setConversationContext()` keeps the tool's JSON Schema clean — the `task`
parameter is the only argument the model sees. `AbstractClient` calls the setter
before `execute()` for `FULL_HISTORY` agents, injecting the conversation
messages externally.

```php
use WebFiori\Ai\Tool\AgentTool;
use WebFiori\Ai\Tool\AgentMessageStrategy;
use WebFiori\Ai\Message;

$agent = new AgentTool(
    name: 'context_aware_agent',
    description: 'An agent that needs full conversation context.',
    provider: $provider,
    profile: $profile,
    messageStrategy: AgentMessageStrategy::FULL_HISTORY,
);

// AbstractClient does this internally before execute():
$agent->setConversationContext([
    new Message('user', 'I am working on a Laravel app.'),
    new Message('assistant', 'Got it. How can I help?'),
    new Message('user', 'I need help with Eloquent queries.'),
]);

// Tool schema stays clean — model only sees { "task": "..." }
$agent->getParameters();
// → { "type": "object", "properties": { "task": { "type": "string", ... } }, "required": ["task"] }
```

**7. Memory integration**

AgentTool optionally accepts `AgentMemory` and `RememberStrategyInterface` for
persistent learning. When memory is configured, `execute()` recalls relevant
memories before calling the provider (injected into the system prompt as
`## Relevant Knowledge`) and remembers new facts after the response via the
strategy.

```php
use WebFiori\Ai\Tool\AgentTool;
use WebFiori\Ai\Tool\AgentProfile;
use WebFiori\Ai\Tool\AgentMemory;
use WebFiori\Ai\Tool\KeywordRememberStrategy;

$agent = new AgentTool(
    name: 'code_reviewer',
    description: 'Reviews code for bugs and security issues.',
    provider: $openAIClient,
    profile: new AgentProfile(identity: 'You are a senior PHP code reviewer.'),
    memory: $memory,                                // AgentMemory instance
    rememberStrategy: new KeywordRememberStrategy(), // extracts facts to store
);

// Getters/setters also available:
$agent->getMemory();
$agent->setMemory($memory);
$agent->getRememberStrategy();
$agent->setRememberStrategy(new KeywordRememberStrategy());
```

**8. Profile inheritance via `extends` and `inheritance_strategy`**

Profiles can inherit from a base profile using an `extends` key (the stem name
of a sibling JSON file, without the `.json` extension). Inheritance resolves
recursively, supports multi-level chains (A → B → C), and detects circular
references (throwing `RuntimeException`). Merging is field-aware and controlled
by two strategies:

- **`merge`** — combine base and child. List fields (`skills`, `instructions`,
  `constraints`, `examples`, `tools`) append; `metadata` merges by key (child
  keys override); string-or-array fields (`context`, `output_format`)
  concatenate, normalizing a plain string to a single-element list first.
- **`replace`** — the child value wins wholesale, falling back to the base when
  the child value is absent, `null`, or empty.

Per-field defaults:

| Field | Default strategy |
|-------|------------------|
| `identity` | `replace` (locked — cannot be overridden) |
| `output_format` | `replace` |
| `context` | `merge` |
| `skills`, `instructions`, `constraints`, `examples`, `tools` | `merge` |
| `metadata` | `merge` (by key) |

`identity` is always `replace` — a single cohesive statement rarely benefits
from concatenation, so it is not overridable via `inheritance_strategy`.
`output_format` defaults to `replace` (preserving prior behavior) but, because
it accepts arrays, may opt into `merge` to accumulate rules across a hierarchy —
giving it full parity with `context`.

```json
{
    "extends": "support-base",
    "inheritance_strategy": {
        "skills": "replace",
        "output_format": "merge"
    },
    "identity": "You are a tier-1 support agent.",
    "skills": ["Only these skills"],
    "output_format": ["Additionally, end with a ticket reference."]
}
```

The strategy vocabulary was deliberately reduced to two verbs (`merge` /
`replace`). An earlier iteration exposed a third strategy, `concat`, that was
functionally identical to `merge` for list fields; collapsing them removes a
distinction without a difference. `concat` is still accepted as a **deprecated
alias** for `merge` (mapped silently) to avoid breaking existing profiles, and
is slated for removal in a future major version.

`extends` and `inheritance_strategy` are resolution-time directives only — they
are stripped from the resolved profile and never appear in `toArray()` /
`toJson()` output.

## Alternatives Considered

**New AgentInterface instead of ToolInterface:**
Would require changes to all provider formatters and response parsers to
recognize a new interface type. Since `ToolInterface` already participates in
the tool loop, there is no benefit to a separate interface. Rejected.

**Pass context as hidden tool argument:**
Adding a `_conversation_context` parameter to the tool schema pollutes the
schema that gets sent to the model, may confuse providers, and breaks
compatibility with strict schema validation. Rejected.

**Always pass full history:**
Wastes tokens for stateless agents (translators, formatters, single-task
specialists) that don't need prior context. The strategy enum gives developers
explicit control over token budget vs. context awareness. Rejected.

**Inline system prompt only:**
A plain string system prompt is not structured, not loadable from external
sources, and not inspectable for testing or debugging. `AgentProfile` provides
structured fields while still supporting plain strings via `fromString()`.
Rejected.

**Agent profile in code only:**
Profiles defined solely in PHP code cannot be changed at runtime without
redeployment. JSON and URL loading enables config-driven agents managed by
platform teams, A/B tested, or versioned externally. Rejected.

## Consequences

**Easier:**
- Multi-agent architectures are composable with existing tool infrastructure
- Agent behavior is configurable without code changes via JSON profiles
- Works with `FallbackProvider`, `ModelRouter`, and `RecordingHttpClient`
- Each agent can use a different provider/model optimized for its task
- Few-shot examples in profiles improve agent accuracy on specialized tasks

**Harder:**
- Nested tool loops increase latency and token usage
- Debugging multi-agent flows requires good observability (logging, tracing)
- `FULL_HISTORY` can be expensive for long conversations
