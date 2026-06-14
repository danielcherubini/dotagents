# Architecture Review Guide

Architecture design review guide to help evaluate whether code architecture is sound and design decisions are appropriate.

## SOLID Principles Checklist

### S - Single Responsibility Principle (SRP)

**Checkpoints:**
- Does this class/module have only one reason to change?
- Do all methods in the class serve the same purpose?
- Could you describe what this class does in one sentence to a non-technical person?

**Signals in code review:**
```
⚠️ Class names contain generalized terms like "And", "Manager", "Handler", "Processor"
⚠️ A class exceeds 200-300 lines of code
⚠️ A class has more than 5-7 public methods
⚠️ Different methods operate on completely different data
```

**Review questions:**
- "What responsibilities does this class have? Could it be split?"
- "If requirement X changes, which methods need to change? What about requirement Y?"

### O - Open/Closed Principle (OCP)

**Checkpoints:**
- Does adding new features require modifying existing code?
- Can new behavior be added through extension (inheritance, composition)?
- Are there large numbers of if/else or switch statements handling different types?

**Signals in code review:**
```
⚠️ Switch/if-else chains handling different types
⚠️ Adding new features requires modifying core classes
⚠️ Type checks (instanceof, typeof) scattered throughout the code
```

**Review questions:**
- "If we need to add a new X type, which files would need to change?"
- "Will this switch statement grow with each new type?"

### L - Liskov Substitution Principle (LSP)

**Checkpoints:**
- Can subclasses fully replace their parent class?
- Do subclasses change the expected behavior of parent class methods?
- Are there subclasses throwing exceptions not declared by the parent?

**Signals in code review:**
```
⚠️ Explicit type casting
⚠️ Subclass methods throwing NotImplementedException
⚠️ Subclass methods as empty implementations or only returning values
⚠️ Code using base classes needs to check specific types
```

**Review questions:**
- "If we replace the parent class with a subclass, do calling code changes?"
- "Does this method's behavior in the subclass match the parent's contract?"

### I - Interface Segregation Principle (ISP)

**Checkpoints:**
- Are interfaces small and focused?
- Are implementing classes forced to implement unnecessary methods?
- Do clients depend on methods they don't use?

**Signals in code review:**
```
⚠️ Interfaces exceeding 5-7 methods
⚠️ Implementing classes with empty methods or throwing NotImplementedException
⚠️ Interface names too broad (IManager, IService)
⚠️ Different clients only use parts of the interface
```

**Review questions:**
- "Are all methods on this interface used by every implementation?"
- "Could we split this large interface into smaller, dedicated ones?"

### D - Dependency Inversion Principle (DIP)

**Checkpoints:**
- Do high-level modules depend on abstractions rather than concrete implementations?
- Is dependency injection used instead of directly new-ing objects?
- Are abstractions defined by high-level modules rather than low-level ones?

**Signals in code review:**
```
⚠️ High-level modules directly new-ing low-level module concrete classes
⚠️ Importing concrete implementation classes instead of interfaces/abstract classes
⚠️ Configuration and connection strings hardcoded in business logic
⚠️ Difficult to write unit tests for a class
```

**Review questions:**
- "Can this class's dependencies be mocked in tests?"
- "How many places would need to change if we switched the database/API implementation?"

---

## Architecture Anti-Patterns

### Critical Anti-Patterns

| Anti-Pattern | Signals | Impact |
|--------------|---------|--------|
| **Big Ball of Mud** | No clear module boundaries, any code can call anything | Hard to understand, modify, and test |
| **God Object** | A single class takes on too many responsibilities, knows too much, does too much | High coupling, hard to reuse and test |
| **Spaghetti Code** | Chaotic control flow, goto or deep nesting, hard to trace execution path | Hard to understand and maintain |
| **Lava Flow** | Ancient code nobody dares touch, lacking documentation and tests | Technical debt accumulates |

### Design Anti-Patterns

| Anti-Pattern | Signals | Recommendation |
|--------------|---------|----------------|
| **Golden Hammer** | Using the same technology/pattern for every problem | Choose solutions appropriate to the problem |
| **Over-Engineering** | Using complex solutions for simple problems, abusing design patterns | Follow YAGNI principle, start simple then grow complex |
| **Boat Anchor** | Unused code written "for when we might need it" | Delete unused code, write it when needed |
| **Copy-Paste Programming** | Same logic appearing in multiple places | Extract common methods or modules |

### Review examples:

```markdown
🔴 [blocking] "This class has 2000 lines of code, recommend splitting into multiple focused classes"
🟡 [important] "This logic is duplicated in 3 places, consider extracting to a common method?"
💡 [suggestion] "This switch statement could be replaced with the Strategy pattern for easier extension"
```

---

## Coupling and Cohesion Assessment

### Coupling Types (best to worst)

| Type | Description | Example |
|------|-------------|---------|
| **Message coupling** ✅ | Passing data through parameters | `calculate(price, quantity)` |
| **Data coupling** ✅ | Sharing simple data structures | `processOrder(orderDTO)` |
| **Stamp coupling** ⚠️ | Sharing complex data structures but using only parts | Passing entire User object but only using name |
| **Control coupling** ⚠️ | Passing control flags that affect behavior | `process(data, isAdmin=true)` |
| **Common coupling** ❌ | Sharing global variables | Multiple modules reading/writing the same global state |
| **Content coupling** ❌ | Directly accessing another module's internals | Directly manipulating another class's private attributes |

### Cohesion Types (best to worst)

| Type | Description | Quality |
|------|-------------|---------|
| **Functional cohesion** | All elements accomplish a single task | ✅ Best |
| **Sequential cohesion** | Output serves as input for the next step | ✅ Good |
| **Communicational cohesion** | Operations on the same data | ⚠️ Acceptable |
| **Temporal cohesion** | Tasks executed at the same time | ⚠️ Poor |
| **Logical cohesion** | Logically related but functionally different | ❌ Bad |
| **Coincidental cohesion** | No obvious relationship | ❌ Worst |

### Metric Reference

```yaml
Coupling Metrics:
  CBO (Class Bonding):
    Good: < 5
    Warning: 5-10
    Critical: > 10

  Ce (Efferent Coupling):
    Description: How many external classes it depends on
    Good: < 7

  Ca (Afferent Coupling):
    Description: How many classes depend on it
    High value means: Changes have wide impact, needs stability

Cohesion Metrics:
  LCOM4 (Lack of Cohesion of Methods):
    1: Single responsibility ✅
    2-3: May need splitting ⚠️
    >3: Should split ❌
```

### Review questions

- "How many other modules does this dependency? Can it be reduced?"
- "How many other places would be affected if we changed this class?"
- "Do all methods in this class operate on the same data?"

---

## Layered Architecture Review

### Clean Architecture Layer Check

```
┌─────────────────────────────────────┐
│         Frameworks & Drivers        │ ← Outermost: Web, DB, UI
├─────────────────────────────────────┤
│         Interface Adapters          │ ← Controllers, Gateways, Presenters
├─────────────────────────────────────┤
│          Application Layer          │ ← Use Cases, Application Services
├─────────────────────────────────────┤
│            Domain Layer             │ ← Entities, Domain Services
└─────────────────────────────────────┘
          ↑ Dependency direction points inward only ↑
```

### Dependency Rules Check

**Core rule: Source code dependencies can only point inward**

```typescript
// ❌ Violation: Domain layer depends on Infrastructure
// domain/User.ts
import { MySQLConnection } from '../infrastructure/database';

// ✅ Correct: Domain defines interfaces, Infrastructure implements
// domain/UserRepository.ts (interface)
interface UserRepository {
  findById(id: string): Promise<User>;
}

// infrastructure/MySQLUserRepository.ts (implementation)
class MySQLUserRepository implements UserRepository {
  findById(id: string): Promise<User> { /* ... */ }
}
```

### Checklist

**Layer boundary checks:**
- [ ] Does the Domain layer have external dependencies (database, HTTP, filesystem)?
- [ ] Does the Application layer directly manipulate the database or call external APIs?
- [ ] Does the Controller contain business logic?
- [ ] Are there cross-layer calls (UI directly calling Repository)?

**Separation of concerns checks:**
- [ ] Is business logic separated from presentation logic?
- [ ] Is data access encapsulated in dedicated layers?
- [ ] Is configuration and environment-related code centrally managed?

### Review examples:

```markdown
🔴 [blocking] "Domain entities directly import database connections, violating dependency rules"
🟡 [important] "Controller contains business calculation logic, recommend moving to Service layer"
💡 [suggestion] "Consider using dependency injection to decouple these components"
```

---

## Design Pattern Usage Assessment

### When to Use Design Patterns

| Pattern | When Appropriate | When Not Appropriate |
|---------|-----------------|---------------------|
| **Factory** | Need to create different types of objects, type determined at runtime | Only one type, or type is fixed |
| **Strategy** | Algorithm needs to switch at runtime, multiple interchangeable behaviors | Only one algorithm, or algorithm won't change |
| **Observer** | One-to-many dependency, state changes need to notify multiple objects | Simple direct calls suffice |
| **Singleton** | Truly need a globally unique instance, like config management | Objects that could be passed via dependency injection |
| **Decorator** | Need to dynamically add responsibilities, avoid inheritance explosion | Responsibilities are fixed, don't need dynamic composition |

### Over-Engineering Warning Signals

```
⚠️ Patternitis signals:

1. Simple if/else replaced with Strategy + Factory + Registry
2. Interfaces with only one implementation
3. Abstract layers added "for when we might need them"
4. Code line count significantly increased by pattern application
5. New team members take a long time to understand the code structure
```

### Review principles

```markdown
✅ Correct pattern usage:
- Solves an actual extensibility problem
- Code is easier to understand and test
- Adding new features becomes simpler

❌ Overuse of patterns:
- Using patterns just for the sake of using them
- Adds unnecessary complexity
- Violates YAGNI principle
```

### Review questions

- "What specific problem does using this pattern solve?"
- "Without this pattern, what would be wrong with the code?"
- "Does the value from this abstraction layer outweigh its complexity?"

---

## Extensibility Assessment

### Extensibility Checklist

**Feature extensibility:**
- [ ] Does adding new features require modifying core code?
- [ ] Are extension points provided (hooks, plugins, events)?
- [ ] Is configuration externalized (config files, environment variables)?

**Data extensibility:**
- [ ] Do data models support new fields?
- [ ] Have you considered data volume growth scenarios?
- [ ] Are there appropriate indexes for queries?

**Load extensibility:**
- [ ] Can it scale horizontally (add more instances)?
- [ ] Are there state dependencies (sessions, local cache)?
- [ ] Is the database connection using a connection pool?

### Extension Point Design Check

```typescript
// ✅ Good extension design: Using events/hooks
class OrderService {
  private hooks: OrderHooks;

  async createOrder(order: Order) {
    await this.hooks.beforeCreate?.(order);
    const result = await this.save(order);
    await this.hooks.afterCreate?.(result);
    return result;
  }
}

// ❌ Poor extension design: Hardcoding all behaviors
class OrderService {
  async createOrder(order: Order) {
    await this.sendEmail(order);        // hardcoded
    await this.updateInventory(order);  // hardcoded
    await this.notifyWarehouse(order);  // hardcoded
    return await this.save(order);
  }
}
```

### Review examples:

```markdown
💡 [suggestion] "If we need to support new payment methods in the future, is this design extensible?"
🟡 [important] "This logic is hardcoded, consider using configuration or strategy patterns?"
📚 [learning] "Event-driven architecture could make this feature easier to extend"
```

---

## Code Structure Best Practices

### Directory Organization

**By feature/domain (recommended):**
```
src/
├── user/
│   ├── User.ts           (entity)
│   ├── UserService.ts    (service)
│   ├── UserRepository.ts (data access)
│   └── UserController.ts (API)
├── order/
│   ├── Order.ts
│   ├── OrderService.ts
│   └── ...
└── shared/
    ├── utils/
    └── types/
```

**By technical layer (not recommended):**
```
src/
├── controllers/     ← Different domains mixed together
│   ├── UserController.ts
│   └── OrderController.ts
├── services/
├── repositories/
└── models/
```

### Naming Convention Check

| Type | Convention | Example |
|------|------------|---------|
| Class names | PascalCase, nouns | `UserService`, `OrderRepository` |
| Method names | camelCase, verbs | `createUser`, `findOrderById` |
| Interface names | I prefix or no prefix | `IUserService` or `UserService` |
| Constants | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| Private attributes | Underscore prefix or private field | `_cache` or `#cache` |

### File Size Guidelines

```yaml
Recommended limits:
  Single file: < 300 lines
  Single function: < 50 lines
  Single class: < 200 lines
  Function parameters: < 4
  Nesting depth: < 4 levels

When exceeding limits:
  - Consider splitting into smaller units
  - Use composition over inheritance
  - Extract helper functions or classes
```

### Review examples:

```markdown
🟢 [nit] "This 500-line file could be split by responsibility"
🟡 [important] "Consider organizing directories by feature domain rather than technical layer"
💡 [suggestion] "Function name `process` is not specific enough, consider `calculateOrderTotal`?"
```

---

## Quick Reference Checklist

### Architecture Review 5-Minute Check

```markdown
□ Are dependency directions correct? (outer layers depend on inner layers)
□ Are there circular dependencies?
□ Is core business logic decoupled from frameworks/UI/database?
□ Does it follow SOLID principles?
□ Are there obvious anti-patterns?
```

### Red Flag Signals (must address)

```markdown
🔴 God Object - Single class exceeding 1000 lines
🔴 Circular dependency - A → B → C → A
🔴 Domain layer contains framework dependencies
🔴 Hardcoded configuration and secrets
🔴 External service calls without interfaces
```

### Yellow Flag Signals (should address)

```markdown
🟡 Class coupling (CBO) > 10
🟡 Method parameters exceeding 5
🟡 Nesting depth exceeding 4 levels
🟡 Duplicated code blocks > 10 lines
🟡 Interfaces with only one implementation
```

---

## Tool Recommendations

| Tool | Purpose | Language Support |
|------|---------|-----------------|
| **SonarQube** | Code quality, coupling analysis | Multi-language |
| **NDepend** | Dependency analysis, architecture rules | .NET |
| **JDepend** | Package dependency analysis | Java |
| **Madge** | Module dependency graphs | JavaScript/TypeScript |
| **ESLint** | Code standards, complexity checks | JavaScript/TypeScript |
| **CodeScene** | Technical debt, hotspot analysis | Multi-language |

---

## Reference Resources

- [Clean Architecture - Uncle Bob](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [SOLID Principles in Code Review - JetBrains](https://blog.jetbrains.com/upsource/2015/08/31/what-to-look-for-in-a-code-review-solid-principles-2/)
- [Software Architecture Anti-Patterns](https://medium.com/@christophnissle/anti-patterns-in-software-architecture-3c8970c9c4f5)
- [Coupling and Cohesion in System Design](https://www.geeksforgeeks.org/system-design/coupling-and-cohesion-in-system-design/)
- [Design Patterns - Refactoring Guru](https://refactoring.guru/design-patterns)
