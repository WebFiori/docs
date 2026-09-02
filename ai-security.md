# Security and PII Redaction

When you log AI requests or emit metrics, message content and headers can contain secrets and personal data. WebFiori AI runs that data through a redaction service before it reaches your logging and metrics callbacks, and you can tune what gets scrubbed.

<meta name="description" content="Protect sensitive data in WebFiori AI logs and metrics with built-in PII redaction, configurable rules, and custom patterns.">

## How Redaction Fits In

Redaction applies to the diagnostic paths, namely logs, metrics, and audit context, not to the request sent to the provider. Its job is to keep credentials and PII out of your observability pipeline. Some patterns, such as API keys and bearer tokens, are always redacted and cannot be turned off.

## Default Behavior

Out of the box, common secrets and PII patterns are redacted, including API keys, bearer tokens, AWS access keys, emails, phone numbers, credit card numbers, and similar identifiers. You get this protection simply by using the logging, metrics, and audit callbacks.

## Configuring Redaction

Attach a `RedactionConfig` to control which built-in rules apply and to add your own. Disabling a rule keeps that data readable in logs (useful for debugging), though the always-on rules ignore any attempt to disable them:

```php
use WebFiori\Ai\Redaction\RedactionConfig;

$client->setRedactionConfig(new RedactionConfig(
    disabledRules: ['email', 'phone'],   // keep these visible for debugging
));
```

## Custom Rules

Add `RedactionRule` entries to redact patterns specific to your domain, such as internal account numbers or a regional ID format:

```php
use WebFiori\Ai\Redaction\RedactionConfig;
use WebFiori\Ai\Redaction\RedactionRule;

$client->setRedactionConfig(new RedactionConfig(
    customRules: [
        new RedactionRule(
            'account_number',
            '/\bACC-\d{8}\b/',
            '[ACCOUNT]'
        ),
    ],
));
```

Each rule has a name, a regular-expression pattern, and the replacement text used when it matches.

## Redacting Request Bodies

By default, full request bodies are not written to logs. If your setup logs bodies and you want them scrubbed too, enable body redaction:

```php
$client->setRedactionConfig(new RedactionConfig(
    redactRequestBodies: true,
));
```

## Good Practices

Keep secrets out of code by reading them from the environment (see [Configuration](learn/ai-configuration)). Leave the always-on credential rules in place, and use audit logging's default of excluding message content unless you have a specific, compliant reason to include it. When you do include content in audits, keep redaction enabled so PII does not leak into stored logs.

## Where to Next

See [Observability](learn/ai-observability) for the logging, metrics, and audit callbacks that redaction protects.
