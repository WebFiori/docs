# WebFiori Documentation

Documentation repository for WebFiori framework. Any change to this repo in the `main` branch is reflected directly to the website at https://webfiori.com/docs.

This repo currently covers **version 3.x** of the framework.

## Quick Start

- **New to WebFiori?** Start with [Introduction](introduction.md)
- **Ready to install?** Check [Installation Guide](installation.md)
- **Browse all topics:** See [Index](index.md)

## Documentation Structure

| Section | Topics |
|---------|--------|
| Getting Started | Introduction, Installation, Folder Structure, Basic Usage |
| Core Features | Routing, Response, Web Services, Middleware |
| UI Development | Web Pages, UI Package, Themes, i18n |
| Data & Storage | Database, Repository Pattern, Migrations, Sessions, Caching, JSON |
| File Handling | Uploading Files, Streaming Uploads, Resumable Uploads |
| Advanced | Background Tasks, Job Queue, Email, Event Dispatcher, DI, Security |
| Infrastructure | CLI, Health Checks, Logging, Environment Variables |

## Libraries Covered

| Library | Docs |
|---------|------|
| `webfiori/framework` | Routing, Middleware, Sessions, CLI commands |
| `webfiori/database` | Database, Repository, Migrations |
| `webfiori/http` | Web Services, MVC |
| `webfiori/file` | File Uploads (standard, streaming, resumable) |
| `webfiori/cli` | Command Line Interface |
| `webfiori/jsonx` | JSON handling |
| `webfiori/ui` | UI Package, Themes, Web Pages |
| `webfiori/mail` | Sending Emails |

## Versioning

See [VERSIONING.md](VERSIONING.md) for the branching and versioning strategy.

Summary: `main` is always the current version. Old versions are frozen into branches (e.g., `v3`) only when work on the next major begins.

## Contributing

1. Fork this repository
2. Create a branch from `dev`
3. Make your changes
4. Submit a pull request to `dev`

Guidelines:
- Use `> **Since X.Y**` notes for features added in minor releases
- Use `> **Deprecated since X.Y.**` for deprecated features
- Keep code examples minimal and runnable
- Verify class/method names against the actual library source

## Links

- **Framework**: [WebFiori on GitHub](https://github.com/WebFiori/framework)
- **Website**: [webfiori.com](https://webfiori.com)
- **API Docs**: [webfiori.com/docs](https://webfiori.com/docs)
