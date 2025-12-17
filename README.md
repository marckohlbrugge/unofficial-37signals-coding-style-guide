# Unofficial 37signals Coding Style Guide

Transferable Rails patterns and development philosophy extracted from analyzing 265 pull requests in 37signals' Fizzy codebase.

## What This Is

Fizzy is a kanban-style project management app built by 37signals. By analyzing their PRs, we extracted reusable patterns applicable to any Rails application—development philosophy, architectural decisions, and implementation techniques.

## What This Is Not

These are not Fizzy-specific implementation details. We deliberately skipped business logic unique to Fizzy and focused only on patterns you can apply to your own projects.

---

## Table of Contents

### Philosophy & Process
- [Development Philosophy](development-philosophy.md) - Ship/Validate/Refine, vanilla Rails, DHH's review patterns
- [Jason Fried's Patterns](jason-fried-patterns.md) - Product-oriented development, perceived performance
- [Jorge Manrubia's Patterns](jorge-manrubia-patterns.md) - Code review style, architecture decisions

### Frontend
- [Hotwire Patterns](hotwire-patterns.md) - Turbo Frames/Streams, morphing, Stimulus, drag & drop
- [Accessibility](accessibility.md) - ARIA patterns, keyboard navigation, screen readers
- [Mobile](mobile.md) - Responsive CSS, safe area insets, touch optimization
- [Filtering](filtering.md) - Filter objects, URL-based state, Stimulus controllers

### Backend
- [Multi-Tenancy](multi-tenancy.md) - Path-based tenancy, middleware, ActiveJob extensions
- [Background Jobs](background-jobs.md) - Solid Queue patterns, tenant preservation, continuable jobs
- [Caching Patterns](caching-patterns.md) - HTTP caching, fragment caching, invalidation
- [Performance Patterns](performance-patterns.md) - Preloading, N+1 prevention, memoization

### Features
- [ActionCable](actioncable.md) - Multi-tenant WebSockets, broadcast scoping, Solid Cable
- [Notifications](notifications.md) - Time window bundling, user preferences, real-time
- [Webhooks](webhooks.md) - SSRF protection, delinquency tracking, state machines
- [Workflows](workflows.md) - Event-driven state, undoable commands
- [Watching Patterns](watching-patterns.md) - Subscription patterns, toggle UI, cache invalidation
- [Email](email.md) - Multi-tenant mailers, timezone handling
- [AI/LLM Integration](ai-llm.md) - Command pattern, cost tracking, LLM tool patterns

### Infrastructure
- [Security Checklist](security-checklist.md) - XSS, CSRF, SSRF, rate limiting, authorization
- [Observability](observability.md) - Structured logging, Yabeda metrics
- [Configuration](configuration.md) - Environment config, Kamal deployment
- [Active Storage](active-storage.md) - Attachment patterns, variants
- [Action Text](action-text.md) - Sanitizer config, remote images

---

## Source

Analysis based on PR list from: https://www.zolkos.com/2025/12/10/fizzys-pull-requests

## Disclaimer

This is an unofficial guide created by analyzing publicly discussed code patterns. It is not affiliated with or endorsed by 37signals.

## License

MIT
