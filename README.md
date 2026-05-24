# Java Spring Boot Code Review — Claude Skill

> A Claude AI skill that performs **deep, senior-engineer-level code reviews** for Java and Spring Boot projects.  
> Goes beyond syntax — surfaces business logic failures, cross-system ripple effects, transaction risks, and production-breaking bugs that standard linters miss.

---

## What This Is

This is a **Claude Skill** — a structured prompt package that extends Claude's behavior for a specific domain. When installed, Claude uses this skill automatically whenever you share Java or Spring Boot code and ask for a review, audit, or impact analysis.

It is designed to behave like a **senior Java engineer pair-reviewing your code**, not just a linter or style checker.

---

## What It Does

Given any Java or Spring Boot code, Claude will:

- Identify **critical bugs** that must be fixed before deployment
- Trace **ripple effects** — what other services, APIs, consumers, or database tables might break
- Catch **Spring Boot-specific pitfalls**: `@Transactional` misuse, AOP proxy bypasses, lazy loading traps, `@Async` self-invocation, bean scope issues, and more
- Flag **business logic failures**: state machine violations, race conditions on inventory, duplicate payment processing, IDOR vulnerabilities
- Review **data integrity**: N+1 queries, missing locks, cascade misconfigurations, pagination gaps
- Check **API contracts**: broken DTOs, wrong HTTP status codes, PII leaking in responses
- Generate a **pre-merge checklist** tailored to what it found

---

## What It Covers

### Spring Boot Checks
| Area | Examples |
|---|---|
| Transactions | `@Transactional` on private/interface methods, wrong propagation, checked exceptions not rolling back |
| AOP Proxies | Self-invocation bypass for `@Transactional`, `@Async`, `@Cacheable`, `@PreAuthorize` |
| Bean Scopes | Stateful singletons, prototype beans injected into singletons, `@RequestScope` misuse |
| JPA / Hibernate | LazyInitializationException, N+1 queries, `CascadeType.ALL` on `@ManyToOne`, dirty checking |
| Security | IDOR, mass assignment via `@RequestBody` to entity, `@PreAuthorize` bypass, wildcard CORS |
| Async & Scheduling | `@Async` without `@EnableAsync`, unhandled async exceptions, missing distributed lock on `@Scheduled` |
| Caching | Wrong cache key, caching nulls, `@Cacheable` on write methods |
| Configuration | `@Value` on static fields, `@ConfigurationProperties` without validation, circular dependencies |

### Business Logic Checks
| Domain | Examples |
|---|---|
| Payments | `double`/`float` for money, wrong rounding mode, currency mismatch |
| Orders | State machine violations, double-processing on retry, partial failure across multi-step operations |
| Inventory | Race condition on stock decrement, overselling, missing optimistic lock (`@Version`) |
| Users | Privilege escalation, account enumeration, email change without re-verification |
| Notifications | Duplicate sends on retry, notifications fired on failure paths, PII in notification content |
| Audit & Compliance | Missing audit trail, non-append-only audit logs, missing `createdBy`/`updatedBy` from security context |
| Idempotency | Non-idempotent POST endpoints, Kafka/RabbitMQ consumer re-entrancy, scheduled job re-processing |
| Migrations | `NOT NULL` columns without defaults, column renames in rolling deploys, enum value backward compatibility |

---

## Output Format

Every review produces:

```
🔍 Code Review: ClassName

Summary — what it does, which layer, overall safety verdict

🚨 Critical Issues       — must fix before deploy, with concrete code fixes
⚠️  Ripple Effect Risks  — what else might break, with severity (LOW/MEDIUM/HIGH/CRITICAL)
🟡 Warnings              — non-blocking but important
💡 Suggestions           — best practice improvements
✅ What's done well      — positive patterns acknowledged
📋 Pre-Merge Checklist   — auto-generated from findings
```

---

## Example

**Input code:**
```java
@Service
public class OrderService {

    @Autowired
    private OrderRepository orderRepo;

    public void cancelOrder(Long orderId) {
        Order order = orderRepo.findById(orderId).orElseThrow();
        order.setStatus("CANCELLED");
        orderRepo.save(order);
    }
}
```

**What this skill catches:**
- 🚨 No `@Transactional` — if `save()` fails, the in-memory status change is lost with no rollback
- 🚨 No ownership check — any authenticated user can cancel any order by guessing the ID (IDOR vulnerability)
- ⚠️ Direct `setStatus()` bypasses a state machine — a SHIPPED order could be set directly to CANCELLED
- ⚠️ No inventory release triggered on cancellation
- 💡 `"CANCELLED"` should be an enum, not a raw String

---

## Installation

### Requirements
- A Claude account (claude.ai, Claude Pro, Claude for Teams, or Claude Enterprise)
- Access to Claude's Skills feature

### Steps

1. **Download this repository** as a ZIP or clone it:
   ```bash
   git clone https://github.com/kumarprabhashanand/java-springboot-code-review.git
   ```

2. **Open Claude** and navigate to **Settings → Skills** (or your team's skill management panel).

3. **Install the skill** by uploading or pointing to the skill folder containing `SKILL.md`.

4. **Verify** by pasting any Java class and asking:  
   *"Review this code"* or *"Is this safe to deploy?"*

> **Note:** The exact installation UI may vary depending on your Claude plan and interface version. Refer to [Anthropic's documentation](https://docs.claude.com) for the latest steps.

---

## Repository Structure

```
java-springboot-code-review/
├── SKILL.md                          # Main skill — Claude reads this on every review
└── references/
    ├── spring-pitfalls.md            # Deep reference: Spring Boot anti-patterns
    └── business-impact-patterns.md   # Deep reference: Business logic failure patterns
```

### File Roles

**`SKILL.md`** — The skill entrypoint. Defines Claude's review persona, phase-by-phase analysis process, output format, and when to load reference files. This is what Claude reads to know how to behave.

**`references/spring-pitfalls.md`** — A detailed catalogue of Spring Boot-specific pitfalls: transaction traps, AOP proxy edge cases, bean lifecycle issues, JPA/Hibernate gotchas, security misconfigurations, async/scheduling problems, caching misuse, and configuration errors. Claude loads this during every review.

**`references/business-impact-patterns.md`** — A catalogue of enterprise business logic failure patterns: payment precision errors, order state machine violations, inventory race conditions, user account vulnerabilities, notification duplicate sends, audit trail gaps, idempotency failures, and data migration risks. Claude loads this when reviewing domain-sensitive code.

---

## How It Works (for AI systems and developers)

This skill uses Claude's **progressive disclosure** loading model:

1. The `SKILL.md` frontmatter (name + description) is always in context — it tells Claude when to trigger this skill.
2. The `SKILL.md` body is loaded when the skill triggers — it defines a 4-phase review process: Context Gathering → Full Code Analysis → Structured Output → Follow-up Behavior.
3. The `references/` files are loaded on demand during Phase 2 — they provide deep technical knowledge without inflating context unnecessarily.

The review process is **deterministic and exhaustive by design**: Claude is instructed to confirm it has checked all 9 analysis lenses (2.1–2.9) before writing any output, preventing silent omissions on simple-looking code.

---

## Supported Languages & Frameworks

| Supported | Notes |
|---|---|
| Java 8, 11, 17, 21 | Version-aware — reviews idiomatically for the detected version |
| Spring Boot 2.x | Full support |
| Spring Boot 3.x | Full support — Jakarta EE namespace, new Security config model |
| Spring MVC / REST | Full support |
| Spring Data JPA | Full support |
| Spring Security | Full support |
| Spring Kafka / RabbitMQ | Supported for consumer idempotency and contract checks |

**Not designed for:** Android Java, Maven/Gradle build files, pure algorithm code with no Spring context.

---

## Contributing

Contributions are welcome — especially:
- New patterns in `references/spring-pitfalls.md` or `references/business-impact-patterns.md`
- Additional review lenses for domains not yet covered (e.g., Spring Batch, Spring Cloud)
- Example inputs and expected outputs to help validate the skill

Please open an issue or pull request.

---

## License

MIT License — see [LICENSE](./LICENSE).

---

## Author

Created and maintained by Kumar Prabhash Anand ([gh](https://github.com/kumarprabhashanand), [in](https://www.linkedin.com/in/kumarprabhashanand/), [medium](https://medium.com/@kumarprabhashanand)).  
Reviewed and refined using Claude's skill creator framework.
