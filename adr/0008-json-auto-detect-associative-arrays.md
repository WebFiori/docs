# ADR-0008: Auto-Detect Associative Arrays During JSON Encoding

**Date:** 2026-06-02
**Status:** Accepted

## Context

When an associative array is passed to `addArray()` without explicitly setting `$asObject = true`, the keys are silently discarded and the array is encoded as a JSON indexed array. This differs from `json_encode()` behavior and causes silent data loss.

```php
$data = ['city' => 'Riyadh', 'country' => 'SA'];

// json_encode — auto-detects associative → JSON object
json_encode($data); // {"city":"Riyadh","country":"SA"}

// Current behavior — keys lost
$json = new Json();
$json->addArray('address', $data);
echo $json; // {"address":["Riyadh","SA"]}
```

The `isIndexedArr()` helper already exists and is used during decoding, but not during encoding.

## Decision

Auto-detect associative arrays during encoding. If the array has non-sequential or string keys, automatically treat it as a JSON object. The `$asObject` parameter remains as an explicit override.

```php
// In JsonConverter::arrayToJsonString or Json::addArray:
if (!$asObj && !self::isIndexedArr($array)) {
    $asObj = true;
}
```

After the change:

```php
$json = new Json();
$json->addArray('address', ['city' => 'Riyadh', 'country' => 'SA']);
echo $json; // {"address":{"city":"Riyadh","country":"SA"}}
```

## Alternatives Considered

- **Throw an exception when associative array is passed without `$asObject = true`** — too strict, bad developer experience.
- **Trigger a deprecation notice** — doesn't actually prevent data loss.
- **No change, document the behavior** — leaves a silent data-loss footgun.

## Consequences

- **Breaking change**: code relying on associative arrays being flattened to indexed arrays (unlikely) will see different output. This is a major version change.
- **Aligns encode/decode behavior**: the library already detects associative arrays during decoding; now encoding matches.
- **Matches `json_encode` semantics**: reduces surprise for PHP developers.
- **`$asObject` parameter remains useful**: for forcing indexed arrays to be treated as objects when keys happen to be sequential integers.

## GitHub Issue

[webfiori/json#56](https://github.com/WebFiori/json/issues/56)
