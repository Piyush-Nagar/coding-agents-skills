# DB Adapter — Query Strategy

How we write database access in a Spring Boot service. The goal is **readable, type-safe queries by default**, dropping down to lower-level tools only when the problem genuinely requires it.

Lives inside `db/<tableName>/dao/` and `db/<tableName>/repository/` — see [backend/core/SKILL.md](../../../../core/SKILL.md) for the module layout.

## The three-tier rule

Pick the **highest** tier that solves the problem. Don't skip tiers for convenience.

### Tier 1 — Derived query methods (preferred)

Spring Data generates the query from the method name on the `*Repo` interface.

```java
public interface TestcaseRepo extends JpaRepository<TestcaseDb, UUID> {
    Optional<TestcaseDb> findByIdAndTenantId(UUID id, UUID tenantId);
    List<TestcaseDb> findByStatusAndCreatedAtAfter(Status status, Instant after);
    long countByOwnerId(UUID ownerId);
}
```

- **Why first:** compile-time checked, zero SQL in your code, trivial to review, works for ~70% of real queries.
- **Use when:** the query is a conjunction of a few predicates, maybe an `OrderBy`, maybe a `Top`/`First`.
- **Stop using when:** the method name reads like a sentence (>4 predicates), you need joins across unrelated entities, or you need projections / aggregations.

### Tier 2 — `@Query` annotation (when Tier 1 can't express it)

Hand-written JPQL (or native SQL / Mongo query) attached to a repo method.

```java
public interface TestcaseRepo extends JpaRepository<TestcaseDb, UUID> {
    @Query("""
        select t from TestcaseDb t
         join t.steps s
        where t.tenantId = :tenantId
          and s.status = :status
          and (:tag is null or :tag member of t.tags)
    """)
    List<TestcaseDb> findByTenantAndStepStatus(
        @Param("tenantId") UUID tenantId,
        @Param("status") StepStatus status,
        @Param("tag") String tag
    );
}
```

Mongo equivalent:

```java
@Query("{ 'tenantId': ?0, 'steps.status': ?1 }")
List<TestcaseDb> findByTenantAndStepStatus(UUID tenantId, StepStatus status);
```

- **Why second:** still declarative, still lives next to the repo interface, still reviewable at a glance.
- **Use when:** you need joins, projections, aggregations, `member of` / `exists` subqueries, or a native query for performance.
- **Rules:** always named parameters (`:name`), never string concatenation. Pin to native (`nativeQuery = true`) only when you need a DB-specific feature.

### Tier 3 — Template / Criteria API in a `*Dao` (last resort)

`JdbcTemplate`, `MongoTemplate`, JPA `CriteriaBuilder`, or QueryDSL. Lives in `db/<tableName>/dao/<Entity>Dao.java` behind the `*Repo` interface — **capabilities code never sees it**.

```java
@Component
class TestcaseDao {
    private final MongoTemplate mongo;

    List<TestcaseDb> search(TestcaseSearchCriteria c) {
        var query = new Query();
        query.addCriteria(Criteria.where("tenantId").is(c.tenantId()));
        if (c.status() != null)   query.addCriteria(Criteria.where("status").is(c.status()));
        if (c.tag() != null)      query.addCriteria(Criteria.where("tags").in(c.tag()));
        if (c.after() != null)    query.addCriteria(Criteria.where("createdAt").gte(c.after()));
        query.with(c.pageable());
        return mongo.find(query, TestcaseDb.class);
    }
}
```

- **Why last:** more code, more surface area for bugs, harder to review. Each dynamic `if` is a path a reviewer has to mentally execute.
- **Use ONLY when:** the `WHERE` clause is **genuinely dynamic** — the set of predicates depends on runtime input (search endpoints, admin filters, report generators).
- **Hide it behind the `*Repo` interface.** Capabilities inject `TestcaseRepo`, not `TestcaseDao`. The dao exists so the query strategy can change without touching callers.

## Rules

- **Repositories expose intent, not SQL.** Method names describe what the caller wants (`findActiveByTenant`), not how it's fetched.
- **One query per method.** No method does two round-trips. Use a join, a projection, or two explicit calls.
- **Named parameters only.** `:tenantId`, never `?1`. Positional parameters break silently when the signature changes.
- **Never concatenate user input into a query string.** Not in `@Query`, not in `Criteria`, not in native SQL. Parameterize everything.
- **Return narrow projections for list endpoints.** Fetching a 30-column entity to render 4 fields wastes DB, network, and GC.
- **No business logic in the dao.** Dao builds the query and runs it. Mapping, validation, orchestration live in the service.
- **Pagination is always `Pageable`.** No `List<X> findAll()` on anything that could grow unbounded.
- **Explain-plan native queries.** If you reached for `nativeQuery = true`, you did it for performance — prove it with `EXPLAIN`.

## Red flags

- `@Query` for something a derived method could express. (Noise.)
- A `*Dao` with 20 static methods, one per fixed query. (Those should be `@Query` on the repo.)
- `String.format` or `+` building a query string.
- Capabilities code depending on `MongoTemplate` / `JdbcTemplate` directly. (Should be behind `*Repo`.)
- Any method that returns an unbounded `List<Entity>` to a controller.
