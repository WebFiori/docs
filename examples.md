# Examples

A collection of real-world example projects built with WebFiori Framework v3. Each project demonstrates specific framework features, progressing from simple to complex.

<meta name="description" content="A set of example projects built with WebFiori Framework to help developers learn by example.">

## Repository

All examples are available at [github.com/webfiori/webfiori-examples](https://github.com/webfiori/webfiori-examples).

## Available Examples

### 1. Task Manager REST API

A pure REST API for managing tasks (to-do items). No UI — JSON input/output only. The simplest example and entry point for learning WebFiori v3.

**Covers:** Web Services, Database, Migrations & Seeders, Testing

### 2. Blog Application

A full-stack blog with server-rendered pages, REST API backend, session-based auth, and internationalization.

**Covers:** WebPage + Themes, Sessions, Middleware, i18n, Pagination, Database, Testing

### 3. Support Ticket System

A support ticket system with file attachments, email notifications, background tasks, and rate limiting.

**Covers:** File Uploads, Email Sending, Background Tasks, Rate Limiting Middleware, Testing

### 4. URL Shortener

A URL shortener with caching, custom CLI commands, dynamic redirects, and a health check endpoint.

**Covers:** Caching, CLI Commands, Dynamic Redirects, Testing

### 5. Multi-Tenant Dashboard

An admin dashboard with role-based access control, audit logging, switchable themes, and Swagger API documentation. The capstone example combining nearly every framework feature.

**Covers:** Privileges, Middleware Chain, Audit Logging, Themes, i18n, Email, Background Tasks, CLI Commands, OpenAPI / Swagger, Testing

## Quick Start

Each example follows the same setup pattern:

```bash
cd <example-folder>
composer install
php webfiori add:db-connection
php webfiori migrations:ini --connection=<name>
php webfiori migrations:run --connection=<name> --env=dev
php -S localhost:8080 -t public
composer test
```

## Prerequisites

- PHP 8.1+ with `mysqli` and/or `sqlsrv` extensions
- Composer
- MySQL or MSSQL database
