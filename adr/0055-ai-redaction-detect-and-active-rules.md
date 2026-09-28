# ADR-0055: Redaction Detection API — Active Rules and Offset-Based `detect()`

**Date:** 2026-09-29
**Status:** Accepted

## Context

`RedactionService` can redact strings and context arrays, but it only exposed
`redactString()` / `redactContext()`, which return already-redacted text. The
built-in rule arrays (`$mandatoryRules`, `$optionalRules`) were `private`.

Consumers that need to **detect and report** PII — not just redact it — must know
*where* each match occurred and *which rule* matched (to build findings such as
`{type, start, end, rule, severity}`). With the rules hidden and only redacted
output available, such a consumer had to re-declare the library's regex patterns
locally and run `preg_match_all(..., PREG_OFFSET_CAPTURE)` — duplicating the
built-in rules and risking silent drift whenever the library updates a pattern.

The motivating consumer is an enterprise AI trust layer whose input-trust
detection needs to emit findings for a prompt while reusing the service's
built-in rules (api_key, email, saudi_id, phone, credit_card, ssn, ip, iban) as
the single source of truth.

## Decision

Add a detection capability alongside redaction, built on a shared active-rule
primitive:

1. **`RedactionService::getActiveRules(): RedactionRule[]`** — returns the exact
   set redaction applies: mandatory + enabled-optional (per `RedactionConfig`) +
   custom, in that order. `ensureCompiled()` was refactored to consume this
   method, so redaction and detection can never drift from one definition of
   "active rules."

2. **`RedactionService::detect(string): RedactionMatch[]`** — runs every active
   rule against the text and returns typed matches.

3. **`RedactionMatch`** — an immutable value object carrying `rule` (name),
   `start`, `end`, `value`, and `replacement`, plus `toArray()`. It deliberately
   does **not** carry `type` or `severity`: those are consumer policy, mapped
   from the rule name by the consumer.

Three detection semantics were decided explicitly:

- **Byte offsets.** The built-in patterns are byte-oriented (no `/u` flag), so
  offsets are PCRE-native byte offsets. `start` is inclusive, `end` exclusive, so
  `substr($text, $start, $end - $start)` recovers the match. No character-offset
  conversion is performed.
- **Emit all overlaps.** Every rule's matches are reported, including spans that
  overlap matches from other rules. Detection does not suppress overlaps the way
  single-pass redaction does; overlap resolution is left to the consumer.
- **Sort by start offset** ascending, with ties preserving active-rule order, so
  output is deterministic and easy to fold into findings.

## Alternatives Considered

**Minimal `getActiveRules()` getter only (no `detect()`):** Smaller surface, and
the originally filed issue leaned this way. Rejected as the primary deliverable
because centralizing offset-capture correctness (byte vs char offsets, overlap
handling) in the library serves multiple consumers better than each
reimplementing the `preg_match_all` loop. `getActiveRules()` is still provided as
the public primitive `detect()` is built on.

**Character offsets:** Rejected — would require committing the byte-oriented
built-in patterns to multibyte handling no current consumer requested, and
diverges from PCRE's native behavior.

**Suppressing overlapping matches (first-rule-wins, mirroring redaction):**
Rejected — a "detect and report" consumer wants to see that multiple rules fired;
suppression is a policy the consumer can apply on top.

**Embedding `type`/`severity` in `RedactionMatch`:** Rejected — the library has
no basis to assign severity; that is consumer domain policy.

## Consequences

- Consumers can reuse the built-in rules for detection with no pattern
  duplication, eliminating drift; the service is the single source of truth for
  both redaction and detection.
- `RedactionMatch`'s field set (`rule`, `start`, `end`, `value`, `replacement`)
  and the three semantics (byte offsets, emit-all-overlaps, sort-by-start) become
  public API; changing them later would be a breaking change.
- Detection returning overlapping matches means consumers that want a single
  label per span must resolve overlaps themselves.
- `getActiveRules()` and `ensureCompiled()` now share one definition of the
  active set, so redaction behavior is unchanged and guaranteed consistent with
  detection.
