# Architecture Guides

Use these guides for system shape, data ownership, reliability, security, and durable product boundaries.

- [architecture.md](architecture.md) - Keep persistence, domain logic, and presentation boundaries clear.
- [api-design.md](api-design.md) - Shape APIs around honest async boundaries, idempotency, and operational contracts.
- [business-logic-computation.md](business-logic-computation.md) - Structure calculation-heavy domain logic.
- [data-model-and-schema-design.md](data-model-and-schema-design.md) - Model durable domain concepts and make invalid states hard to represent.
- [data-privacy-and-compliance.md](data-privacy-and-compliance.md) - Minimize sensitive data and make retention explicit.
- [reliability-and-resilience.md](reliability-and-resilience.md) - Make workflows safe under retries, partial failure, and interruption.
- [security-and-authorization.md](security-and-authorization.md) - Enforce trust boundaries and sensitive action protection.
- [transactional-vs-analytical-databases.md](transactional-vs-analytical-databases.md) - Separate source-of-truth state from derived analytical reads.
