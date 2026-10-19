# Spec-Driven Development

Write the API contract **before** the implementation. The spec — OpenAPI / Protobuf / GraphQL SDL — is the single source of truth that producers and consumers agree on. Code is generated from it; it is not reverse-engineered from code.

## Rules

- **Spec first, code second.** No controller gets written before the OpenAPI (or `.proto` / SDL) entry it implements. Prevents "the spec we published" ≠ "the API that shipped."
- **The spec lives in version control.** Beside the code, reviewed in the same PR as the change. If it's in a wiki or Postman, it will drift.
- **Generate code from the spec, don't hand-write it.** Server stubs, DTOs, and client SDKs come from codegen. Hand-written DTOs are how fields silently diverge.
- **One spec per service, owned by that service.** Shared "god spec" repos become unmergeable. Reference shared schemas via `$ref`, don't duplicate them.
- **Review the spec before the implementation.** API design — nouns, verbs, error shape, pagination, idempotency — is cheaper to change in YAML than in prod. Approve the contract separately.
- **Breaking changes mean a new version.** New required field, renamed field, removed endpoint, changed error shape — all breaking. Versioning (`/v2/...`) or additive-only is the only safe path.
- **Validate requests and responses against the spec in CI.** Contract tests on the server (Spring REST Docs, Pact, Dredd) and schema validation on the client. Prevents silent drift.
- **Document errors in the spec too.** Every endpoint lists its 4xx/5xx shapes. Clients can't handle what isn't declared.
- **Spec is the authority on examples.** Keep request/response examples in the spec, not in a separate Confluence page. One source, one truth.
- **Deprecate before deleting.** Mark fields/endpoints `deprecated: true` for at least one release before removal. Gives consumers a defined migration window.

## Red flags

- An OpenAPI file auto-generated from Spring annotations at build time — the code is the source of truth, the spec is just a mirror. (Backward.)
- Frontend engineers "asking what the response looks like" in Slack. The spec should answer that.
- Two services with their own hand-written DTOs for the same payload shape.
- Breaking changes merged without a version bump "because no one's using it yet."
- Spec changes merged in a different PR (and often a different sprint) than the code that implements them.
