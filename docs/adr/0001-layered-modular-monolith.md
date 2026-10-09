# ADR-0001: Layered monolith organised by module

- **Status:** Accepted
- **Date:** 2026-10-09

## Context

Libryx is a portfolio project built by one developer: a REST API for a single library, consumed by an
Angular frontend that lives in its own repository. The domain has a handful of modules (auth, users,
catalog, loans, sanctions, notifications, exports, assistant) with strong consistency needs between
them: a loan, its copy, the request queue and a sanction change together.

## Decision

The API is one Spring Boot application, split first by **business module** and then by **layer**
(`controller → service → repository`). Modules talk to each other only through their service
interfaces; circular dependencies are broken with application events.

## Alternatives considered

| Alternative | Why it was discarded |
|---|---|
| Hexagonal / Clean Architecture | Ports, adapters and use-case classes add indirection with no second adapter to justify it. The author's other portfolio projects already show those styles |
| Microservices | Distributed transactions between loans, copies and sanctions, plus the operational cost, for a single library with one team |
| Package by layer only | Mixes every module in `controller/`, `service/`… and hides the module boundaries |

## Consequences

- **Gains:** simple local development and deployment, real ACID transactions across modules, clear
  module boundaries that could be extracted later.
- **Costs:** one deployable unit; one module's load affects the others.
- **Watch for:** a module that needs to scale or be released on its own, or modules reaching into
  each other's repositories.
