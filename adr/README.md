# Architecture Decision Records

This directory contains Architecture Decision Records (ADRs) for the WebFiori Framework.

## What is an ADR?

An ADR is a short document that captures a significant design decision along with its context and consequences. ADRs are numbered sequentially and are immutable once accepted — if a decision is reversed, a new ADR supersedes the old one.

## Template

Use [0000-template.md](0000-template.md) when creating a new ADR.

## Index

| # | Title | Status | Date |
|---|-------|--------|------|
| 0001 | [Access::can() Role Resolution Strategy](0001-access-can-role-resolution.md) | Accepted | 2026-05-31 |
| 0002 | [Middleware Dependency Auto-Resolution](0002-middleware-dependency-auto-resolution.md) | Accepted | 2026-05-31 |
| 0003 | [#[RequiresAuth] Should Check SecurityContext Directly](0003-requires-auth-security-context.md) | Accepted | 2026-05-31 |
| 0004 | [Column Attribute Name Parameter Must Be Respected](0004-column-name-parameter-respected.md) | Accepted | 2026-05-31 |
| 0005 | [Replace WebServicesManager with RequestProcessor](0005-request-processor-replaces-manager.md) | Accepted | 2026-06-01 |
| 0006 | [ServiceRouter — Framework Integration](0006-service-router-framework-integration.md) | Accepted | 2026-06-01 |
| 0007 | [Validation Errors 422 With Messages](0007-validation-errors-422-with-messages.md) | Accepted | 2026-06-01 |
| 0008 | [Json: Auto-Detect Associative Arrays During Encoding](0008-json-auto-detect-associative-arrays.md) | Proposed | 2026-06-02 |
| 0009 | [Json: Include All Getter Return Values and Add #[JsonIgnore]](0009-json-include-null-false-and-json-ignore.md) | Proposed | 2026-06-02 |
| 0010 | [Json: Normalize Getter-Derived Names and Add #[JsonProperty]](0010-json-normalize-getter-names-and-json-property.md) | Proposed | 2026-06-02 |
| 0011 | [Json: Typed Deserialization With Nested Object Hydration](0011-json-typed-deserialization.md) | Proposed | 2026-06-02 |
| 0012 | [Json: Remove Error Suppression During Getter Calls](0012-json-remove-error-suppression.md) | Proposed | 2026-06-02 |
| 0013 | [Json: Deprecate Global Constants in Favor of Static Defaults](0013-json-deprecate-global-constants.md) | Proposed | 2026-06-02 |
| 0014 | [Json: Serialize INF/NaN as Strings (Never Crash)](0014-json-inf-nan-as-strings.md) | Accepted | 2026-06-02 |
| 0015 | [Json: Only Support `get*` Prefix for Auto-Mapping](0015-json-only-get-prefix.md) | Accepted | 2026-06-02 |
| 0016 | [Introduce FileInterface Abstraction](0016-file-interface-abstraction.md) | Proposed | 2026-06-02 |
| 0017 | [UI: Extract Renderer from HTMLNode](0017-ui-extract-renderer-from-htmlnode.md) | Proposed | 2026-06-03 |
| 0018 | [Collections: Add Vector (Array-Backed, O(1) Index Access)](0018-collections-add-vector.md) | Proposed | 2026-06-03 |
| 0024 | [CLI: Verbosity Levels Without Built-in Logging](0024-cli-verbosity-not-logging.md) | Proposed | 2026-06-13 |
| 0025 | [CLI: Command Metadata via PHP Attributes](0025-cli-attributes-for-metadata.md) | Proposed | 2026-06-13 |
| 0026 | [CLI: ANSI Colors On by Default (TTY Auto-Detection)](0026-cli-ansi-auto-detect.md) | Proposed | 2026-06-13 |
| 0027 | [AI Library as Standalone Package](0027-ai-standalone-package.md) | Accepted | 2026-07-06 |
| 0028 | [AI: Provider-Agnostic Interface Design](0028-ai-provider-agnostic-interface.md) | Accepted | 2026-07-06 |
