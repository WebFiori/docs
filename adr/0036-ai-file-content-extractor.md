# ADR-0036: AI: FileContentExtractor — Universal File-to-Text Tool

**Date:** 2026-08-16
**Status:** Accepted

## Context

AI models cannot process certain binary file formats directly. When a user sends
an `.xlsx`, `.docx`, or `.pptx` file, the model either fails or hallucinates
content. There was no built-in way to convert these files to a format the model
can understand.

Additionally, different providers natively support different file types. Google
can process PDFs and images directly; OpenAI cannot process `.xlsx` at all.
Developers building multi-provider applications had to implement file conversion
themselves for each provider.

## Decision

Add a `FileContentExtractor` tool in `WebFiori\Ai\Tool\FileProcessing\` that
acts as a universal file reader. The model invokes it when it encounters a file
it needs to process. The tool handles detection, conversion, and output
formatting in a single call.

**Architecture:**

```
FileContentExtractor (ToolInterface)
├── FileTypeDetector     — MIME + extension detection via finfo + extension map
├── ConverterRegistry    — maps (mime, extension) → ConverterInterface, with priority
└── Converters
    ├── TextConverter        — text/*, .csv, .json, .xml, .html, code files
    ├── SpreadsheetConverter — .xlsx, .ods via ZipArchive + XML parsing
    ├── DocumentConverter    — .docx, .odt via ZipArchive + XML parsing
    └── PresentationConverter — .pptx via ZipArchive + XML parsing
```

**Converter registration uses extension + MIME with priority:**

```php
$extractor->registerConverter(new MyConverter(), priority: 10);
// Built-ins have priority 0 — developer converters always win
```

**Output format priority (highest to lowest):**
1. Developer `setDefaultOutputFormat()` — locks the format for all calls
2. Model parameter `output_format` in tool arguments
3. Converter's `getDefaultOutputFormat()` — per-format sensible default

**Max output priority (highest to lowest):**
1. Model parameter `max_output` in tool arguments
2. Global `setMaxOutput()` on the tool
3. Built-in default: 50,000 characters (~12,500 tokens)

When content exceeds the limit, it is truncated with a note appended so the
model knows to request a specific range rather than silently receiving
incomplete data.

**Security:** Opt-in path allowlist via `setAllowedPaths()`. When set, any path
outside the allowed directories throws before the file is opened. Disabled by
default to avoid friction for trusted internal use cases.

**Unsupported file types:** Throws `UnsupportedFileTypeException`. The tool
catches it and returns a structured JSON error to the model so the conversation
continues rather than crashing.

**Input:** File path (local), URL (fetched via HTTP), or raw bytes.

## Alternatives Considered

**Multi-tool chain (detect → decide → convert):**
Rejected — 3-4x slower due to multiple API round-trips per file.

**Pre-processing before sending to model:**
Rejected — removes the model's ability to decide when and how to process a
file. The model may not need the full content of every attachment.

**Third-party library (PhpSpreadsheet, etc.):**
Rejected — violates the zero-dependency principle. Office Open XML formats
(`.xlsx`, `.docx`, `.pptx`) are ZIP archives containing XML, parseable with
PHP's built-in `ZipArchive` and `SimpleXML`.

**Silent fallback for unsupported types (return metadata only):**
Rejected — in an AI pipeline, silent failures produce hallucinations. The model
receives partial information and confidently generates wrong answers. Throwing
makes the gap visible immediately.

## Consequences

**Easier:**
- Developers register one tool and send any file — the tool handles the rest
- Custom formats are supported by registering a converter with priority > 0
- Security-conscious deployments can restrict paths with one method call
- The model receives structured JSON with metadata (sheet names, page count,
  truncation status) enabling follow-up calls for specific sections

**Harder:**
- Legacy binary formats (`.doc`, `.xls`) require CLI tools (`antiword`,
  `catdoc`) or a custom converter — not handled by built-ins
- URL input requires an HTTP call inside the tool, adding latency
- Large files with max_output truncation may lose important content; developers
  must tune the limit per use case
