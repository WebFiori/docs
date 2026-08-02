# Job Queue

The job queue allows you to defer time-consuming work to be processed in the background. This is useful for tasks like sending emails, processing payments, generating reports, or any operation that shouldn't block the HTTP response.

<meta name="description" content="Learn how to use the WebFiori job queue for background processing with priority, retry, and encryption.">

## Key Concepts

- **Job** — A unit of work that implements the `Job` interface
- **Queue** — Dispatches jobs and processes them with retry logic
- **QueueStorage** — Pluggable backend (file-based by default)
- **QueueFacade** — Static API for the default queue instance

## Creating a Job

A job is a class that implements `WebFiori\Queue\Job`:

```php
<?php
namespace App\Jobs;

use WebFiori\Queue\Job;

class SendWelcomeEmailJob implements Job {
    public function __construct(
        private string $email,
        private string $name
    ) {
    }

    public function getMaxAttempts(): int {
        return 3;
    }

    public function getRetryDelaySeconds(): int {
        return 30;
    }

    public function handle(): void {
        // Send the email
        $message = new \WebFiori\Mail\Email(new \WebFiori\Mail\SMTPAccount());
        $message->setSubject('Welcome!');
        $message->addTo($this->email, $this->name);
        $message->insert('p')->text("Welcome, {$this->name}!");
        $message->send();
    }
}
```

The `Job` interface requires three methods:

| Method | Purpose |
|--------|---------|
| `handle()` | The work to perform |
| `getMaxAttempts()` | How many times to retry on failure |
| `getRetryDelaySeconds()` | Base delay between retries (multiplied by attempt number) |

## Dispatching Jobs

Use `QueueFacade` to dispatch jobs:

```php
use App\Jobs\SendWelcomeEmailJob;
use WebFiori\Queue\QueueFacade;

// Basic dispatch
QueueFacade::dispatch(new SendWelcomeEmailJob('user@example.com', 'Alice'));

// With priority (higher = processed first)
QueueFacade::dispatch(new SendWelcomeEmailJob('vip@example.com', 'Bob'), priority: 10);

// With delay (process after 60 seconds)
QueueFacade::dispatch(new SendWelcomeEmailJob('user@example.com', 'Carol'), delaySeconds: 60);
```

## Processing Jobs

Jobs are processed by calling `QueueFacade::process()`. This is typically done via a long-running CLI process:

```bash
# Process up to 10 jobs per cycle, continuously
php -r "
require 'vendor/autoload.php';
\WebFiori\Framework\App::initiate('App', 'public', __DIR__.'/public');
\WebFiori\Framework\App::start();
while (true) {
    \WebFiori\Queue\QueueFacade::process(10);
    sleep(1);
}
"
```

Or create a CLI command for it:

```php
// In a custom command
QueueFacade::process(10); // Process up to 10 pending jobs
```

The `process()` method:
1. Retrieves available jobs from storage (respects `availableAt` and priority)
2. Deserializes and decrypts the job payload
3. Calls `handle()`
4. On success: marks the job as complete
5. On failure: retries with exponential backoff, or moves to failed queue after max attempts

## Retry Logic

When a job throws an exception:

1. If `attempts < getMaxAttempts()`: re-queued with delay = `getRetryDelaySeconds() × attempts`
2. If `attempts >= getMaxAttempts()`: moved to the failed queue with the error message

Example: a job with `getMaxAttempts() = 3` and `getRetryDelaySeconds() = 30`:
- Attempt 1 fails → retry after 30s
- Attempt 2 fails → retry after 60s
- Attempt 3 fails → moved to failed queue

## Failed Jobs

```php
// Get all failed jobs
$failed = QueueFacade::getFailed();

foreach ($failed as $queuedJob) {
    echo $queuedJob->getId() . ': ' . $queuedJob->getFailReason() . "\n";
}

// Retry a specific failed job
QueueFacade::retry($jobId);

// Clear all failed jobs
QueueFacade::flush();
```

## Payload Encryption

Set the `QUEUE_KEY` environment variable to encrypt job payloads at rest using AES-256-GCM:

```json
"env-vars": {
    "QUEUE_KEY": {
        "value": "env:QUEUE_ENCRYPTION_KEY",
        "description": "AES-256-GCM key for queue payload encryption."
    }
}
```

When set, all job payloads are encrypted before storage and decrypted during processing. When not set, payloads are stored as plain serialized PHP.

## Storage Backends

The default storage is `FileQueueStorage`, which stores jobs as files in a directory. You can implement `QueueStorage` for other backends (Redis, database, etc.):

```php
use WebFiori\Queue\QueueStorage;
use WebFiori\Queue\QueuedJob;

class RedisQueueStorage implements QueueStorage {
    public function push(QueuedJob $job): void { /* ... */ }
    public function pop(int $limit = 10): array { /* ... */ }
    public function markComplete(string $id): void { /* ... */ }
    public function markFailed(QueuedJob $job): void { /* ... */ }
    public function getFailed(): array { /* ... */ }
    public function retry(string $id): void { /* ... */ }
    public function flush(): void { /* ... */ }
    public function getPendingCount(): int { /* ... */ }
}
```

If your backend supports listing all pending jobs (files, database, Redis), implement `ListableQueueStorage` instead:

```php
use WebFiori\Queue\ListableQueueStorage;
use WebFiori\Queue\QueuedJob;

class RedisQueueStorage implements ListableQueueStorage {
    // ... all QueueStorage methods plus:

    public function getPending(): array {
        // Return all pending jobs sorted by priority desc, createdAt asc
    }
}
```

Backends that cannot enumerate their contents (SQS, RabbitMQ) should only implement the base `QueueStorage` interface.

To use a custom storage:

```php
use WebFiori\Queue\Queue;
use WebFiori\Queue\QueueFacade;

$queue = new Queue(new RedisQueueStorage($redis));
QueueFacade::setInstance($queue);
```

## Checking Queue Status

```php
// Number of pending jobs (works with any storage backend)
$count = QueueFacade::getPendingCount();

// List all pending jobs including delayed ones (requires ListableQueueStorage)
$pending = QueueFacade::getPending();

foreach ($pending as $queuedJob) {
    echo $queuedJob->getId() . ' | '
        . 'priority=' . $queuedJob->getPriority() . ' | '
        . 'attempts=' . $queuedJob->getAttempts() . ' | '
        . 'available=' . date('Y-m-d H:i:s', $queuedJob->getAvailableAt()) . "\n";
}
```

> **Note:** `getPending()` requires a `ListableQueueStorage` backend (e.g. `FileQueueStorage`). If the storage backend does not support listing (e.g. an SQS adapter), a `LogicException` is thrown. Use `getPendingCount()` as a universal alternative for simple checks.

## Best Practices

- Keep jobs small and focused — one job, one task
- Make jobs idempotent — safe to retry without side effects
- Set appropriate `getMaxAttempts()` based on the failure mode (transient vs permanent)
- Use priority to ensure critical jobs (payments) are processed before low-priority ones (emails)
- Use delayed dispatch for rate-limited external APIs
- Monitor the failed queue and set up alerts
