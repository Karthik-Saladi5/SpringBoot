
# Spring Boot Internals — What I Learned

## 1. Spring is an IoC Container

Before Spring:

```java
StudentRepository repo = new StudentRepository();
StudentService service = new StudentService(repo);
```

With Spring:

```text
Spring creates objects
Spring manages objects
Spring wires objects together
```

Spring acts as an **Object Factory + Dependency Manager**.

---

# 2. Bean Definition ≠ Bean

### Bean Definition

A blueprint/metadata for creating a bean.

Observed at:

```java
registerBeanDefinition(...)
```

At this stage:

```text
StudentService is known to Spring
but object does not exist yet
```

---

### Bean

Actual object in memory.

Observed at:

```java
createBeanInstance(...)
```

At this stage:

```text
StudentService object is created
```

---

# 3. Application Startup Flow

```text
Application Starts
        ↓
Component Scan
        ↓
Register Bean Definitions
        ↓
Create Repository Beans
        ↓
Create Service Beans
        ↓
Inject Dependencies
        ↓
@PostConstruct
        ↓
Create Proxies (if needed)
        ↓
Store Beans in IoC Container
        ↓
Application Ready
```

---

# 4. Dependency Injection

Spring builds dependencies from the bottom up.

Example:

```text
StudentController
        ↓
StudentService
        ↓
StudentRepository
```

Creation order:

```text
Repository
↓
Service
↓
Controller
```

I verified this using constructor breakpoints.

---

# 5. Constructor Injection

Example:

```java
public StudentService(StudentRepository repository)
```

Spring effectively does:

```java
new StudentService(repository);
```

What I observed:

```text
repository parameter already populated
```

Meaning:

```text
Dependency resolved before constructor executes
```

---

# 6. Bean Lifecycle

Observed through:

```java
@PostConstruct
```

Flow:

```text
Constructor
        ↓
Dependency Injection
        ↓
@PostConstruct
        ↓
Bean Ready
```

Key insight:

```text
Object creation
≠
Bean initialization
```

---

# 7. Repository is a Proxy

I saw:

```text
$Proxy153
```

instead of:

```text
StudentRepositoryImpl
```

Meaning:

```text
StudentRepository interface
        ↓
Spring generates implementation
        ↓
Injects proxy
```

Spring Data JPA creates repository implementations dynamically.

---

# 8. Why Spring Uses Proxies

Spring wants to run logic:

```text
Before Method
↓
Method
↓
After Method
```

without modifying my code.

Instead of:

```text
Controller
↓
Service
```

actual flow is:

```text
Controller
↓
Proxy
↓
Real Service
```

---

# 9. How @Transactional Works

I learned:

```java
@Transactional
```

does NOT create transactions.

Actual flow:

```text
@Transactional
        ↓
Spring detects annotation
        ↓
Creates proxy
        ↓
Proxy starts transaction
        ↓
Calls service method
        ↓
Commits/Rolls back
```

The proxy does the work.

---

# 10. Why Self Invocation Fails

Example:

```java
public void methodA() {
    methodB();
}

@Transactional
public void methodB() {
}
```

Inside the same class:

```java
this.methodB();
```

bypasses the proxy.

Result:

```text
No transaction starts
```

Because:

```text
Proxy not involved
```

---

# 11. Why Constructor Injection is Preferred

Not because field injection doesn't work.

Field injection:

```java
@Autowired
private StudentRepository repository;
```

Flow:

```text
Create object
↓
Inject dependency later
```

Constructor injection:

```java
public StudentService(StudentRepository repository)
```

Flow:

```text
Create object with dependency
```

Benefits:

```text
Dependencies are explicit
Object is always valid
Supports final fields
Easier testing
Detects circular dependencies early
```

---

# 12. My Current Mental Model

### Startup

```text
Scan
↓
Register Beans
↓
Create Objects
↓
Inject Dependencies
↓
@PostConstruct
↓
Create Proxies
↓
Store in Container
```

### Request Flow

```text
HTTP Request
↓
Controller
↓
Proxy
↓
Transaction Starts
↓
Service
↓
Repository Proxy
↓
Hibernate
↓
Database
↓
Commit Transaction
↓
Response
```

---

# Biggest Realization

Before:

```text
Spring feels like magic.
```

Now:

```text
Spring is mostly:
- Bean creation
- Dependency injection
- Lifecycle management
- Proxy generation
```

The "magic" is actually a series of understandable steps that can be observed in the debugger.

---

### Next Topic to Learn

The next major concept is:

```text
Hibernate / JPA Internals
        ↓
Persistence Context
        ↓
Managed Entities
        ↓
Dirty Checking
        ↓
Flush
```

This explains why:

```java
student.setName("John");
```

can update the database even without:

```java
repository.save(student);
```

which is the next piece of Spring/JPA magic to demystify. 🚀
