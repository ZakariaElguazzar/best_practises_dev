---

## Validation Strategy: Boundary-Driven Validation

### 1. The Core Philosophy

Validation should not be the responsibility of the Entity's setters. Instead, validation should occur at the **Application Boundaries** (Controllers or Service Layer) using Data Transfer Objects (DTOs). This prevents "Polluted Entities" and ensures that invalid data never reaches the core of your system.

### 2. Implementation Layers

#### A. The API Boundary (DTO Validation)

Use dedicated DTOs for incoming requests. This allows you to apply strict validation rules that might differ between "Creating" a patient and "Updating" a patient.

```java
// PatientRequestDTO.java
public record PatientRequestDTO(
    @NotBlank(message = "Name is required")
    String name,
    
    @Email(message = "Invalid email format")
    String email,
    
    @Past(message = "Birth date must be in the past")
    LocalDate dateOfBirth
) {}

```

#### B. The Controller Level

Use the `@Valid` or `@Validated` annotation to trigger the validation engine before your business logic even starts.

```java
@PostMapping
public ResponseEntity<Patient> createPatient(@Valid @RequestBody PatientRequestDTO dto) {
    // If validation fails, a 400 Bad Request is returned automatically
    return ResponseEntity.ok(patientService.register(dto));
}

```

### 3. Why This is Best Practice

* **Decoupling:** Your `Patient` entity doesn't need to know about web-specific validation messages or constraints.
* **Fail-Fast:** By validating at the Controller, you stop invalid requests before they consume database connections or CPU cycles in the Service layer.
* **Immutable-Friendly:** This approach works perfectly with `records` or `@Builder`, where objects are often created once rather than modified via setters.
* **Security:** Using DTOs prevents **Mass Assignment Vulnerabilities**, ensuring users can only update fields you explicitly expose.

### 4. Comparison Table

| Feature | Entity-Level Validation | Boundary (DTO) Validation |
| --- | --- | --- |
| **Primary Goal** | Database Integrity | User Input Integrity |
| **Flexibility** | Rigid (One set of rules) | High (Different rules per API endpoint) |
| **Performance** | Late (Checks at save time) | Early (Checks at request time) |
| **Complexity** | Simple for small apps | Scalable for enterprise apps |

---

> **Pro-Tip:** Even when using Boundary Validation, keep `@Column(nullable = false, unique = true)` on your JPA Entities. This acts as a **"Safety Net"** at the database schema level, even if the application validation is somehow bypassed.

Below are **Spring Boot best practices** presented in the same structured format you requested.

---

## Transaction Management: Service-Layer Boundaries

### 1. The Core Philosophy

Transactions should be managed at the **Service layer**, not in Controllers or Repositores. The service layer defines the business use case boundary, ensuring data consistency and preventing partial updates.

### 2. Implementation Layers

#### A. Service Layer Transaction Boundary

Annotate service methods with `@Transactional` to ensure atomic operations.

```java
@Service
public class PatientService {

    @Transactional
    public void updatePatient(Long id, String name) {
        Patient patient = repository.findById(id).orElseThrow();
        patient.setName(name); // Dirty checking will persist change
    }
}
```

#### B. Read-Only Optimization

Use read-only transactions for queries.

```java
@Transactional(readOnly = true)
public List<Patient> findAll() {
    return repository.findAll();
}
```

### 3. Why This is Best Practice

* **Consistency:** Ensures all operations succeed or fail together.
* **Performance:** Read-only transactions reduce overhead.
* **Separation of Concerns:** Controllers remain focused on HTTP logic.
* **Reliability:** Prevents partial data persistence.

### 4. Comparison Table

| Location   | Recommended | Reason                             |
| ---------- | ----------- | ---------------------------------- |
| Controller | ❌           | Not a business boundary            |
| Service    | ✅           | Defines use-case transaction scope |
| Repository | ❌           | Too granular                       |

---

## DTO Mapping Strategy: Protecting Domain Integrity

### 1. The Core Philosophy

Never expose JPA entities directly through APIs. Use DTOs to control data flow and protect internal structures.

### 2. Implementation Layers

#### A. Response DTO

```java
public record PatientDTO(Long id, String name, String email) {}
```

#### B. Mapping in Service Layer

```java
public PatientDTO toDTO(Patient patient) {
    return new PatientDTO(
        patient.getId(),
        patient.getName(),
        patient.getEmail()
    );
}
```

### 3. Why This is Best Practice

* Prevents overexposing sensitive fields
* Avoids lazy loading serialization errors
* Allows versioning of APIs
* Supports different views of the same entity

### 4. Comparison Table

| Approach        | Risk | Flexibility |
| --------------- | ---- | ----------- |
| Entity Exposure | High | Low         |
| DTO Mapping     | Low  | High        |

---

## Persistence Performance: Lazy Loading & Query Optimization

### 1. The Core Philosophy

Load only the data you need to avoid performance bottlenecks and memory overhead.

### 2. Implementation Layers

#### A. Prefer Lazy Loading

```java
@ManyToOne(fetch = FetchType.LAZY)
private Doctor doctor;
```

#### B. Avoid N+1 Queries

```java
@Query("SELECT p FROM Patient p JOIN FETCH p.doctor")
List<Patient> findAllWithDoctor();
```

### 3. Why This is Best Practice

* Reduces unnecessary database load
* Prevents performance degradation at scale
* Improves response times

### 4. Comparison Table

| Strategy      | Performance   | Use Case            |
| ------------- | ------------- | ------------------- |
| EAGER Loading | Poor at scale | Small object graphs |
| LAZY Loading  | Excellent     | Enterprise systems  |

---

## Exception Handling: Centralized Error Management

### 1. The Core Philosophy

Handle exceptions globally to ensure consistent API responses and prevent leaking internal details.

### 2. Implementation Layers

#### A. Global Exception Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(PatientNotFoundException.class)
    public ResponseEntity<String> handleNotFound() {
        return ResponseEntity.status(404).body("Patient not found");
    }
}
```

### 3. Why This is Best Practice

* Consistent error responses
* Prevents sensitive information leaks
* Simplifies controller code
* Improves API consumer experience

### 4. Comparison Table

| Approach             | Maintainability | Security |
| -------------------- | --------------- | -------- |
| Try/catch everywhere | Poor            | Risky    |
| Global Handler       | Excellent       | Safe     |

---

## Configuration Management: Environment Isolation

### 1. The Core Philosophy

Applications must behave differently across environments (dev, test, prod). Configuration should be externalized and environment-specific.

### 2. Implementation Layers

#### A. Use Profiles

```properties
spring.profiles.active=dev
```

#### B. Profile-Specific Files

```
application-dev.properties
application-prod.properties
```

### 3. Why This is Best Practice

* Prevents production misconfiguration
* Secures credentials
* Simplifies deployments
* Supports CI/CD pipelines

### 4. Comparison Table

| Strategy                   | Risk | Scalability |
| -------------------------- | ---- | ----------- |
| Single config file         | High | Low         |
| Profiles & external config | Low  | High        |

---

> **Pro Tip:** Combine DTO validation, service-layer transactions, and centralized exception handling to create a **fail-fast, secure, and production-ready Spring Boot architecture**.

---

Voici les **bonnes pratiques essentielles** pour développer des applications robustes avec **Spring Boot**. Elles couvrent l’architecture, la persistance, la performance et la maintenabilité.

---

# 🧱 1. Architecture & Organisation du Code

## ✅ Utiliser une architecture en couches claire

```
controller → service → repository → database
```

### ✔ Règles

* **Controller** → gestion HTTP uniquement
* **Service** → logique métier
* **Repository** → accès aux données
* **DTO** → échanges API

👉 Évite la logique métier dans les controllers.

---

## ✅ Structurer par fonctionnalité (feature-based)

✔ recommandé pour les projets évolutifs

```
patient/
 ├── PatientController
 ├── PatientService
 ├── PatientRepository
 ├── dto/
 └── entity/
```

---

# 🔄 2. Gestion des Transactions

## ✅ Placer `@Transactional` dans la couche service

✔ bonne pratique :

```java
@Service
@Transactional
public class PatientService { }
```

❌ éviter dans les controllers

---

## ✅ Utiliser readOnly pour les lectures

```java
@Transactional(readOnly = true)
public List<Patient> findAll() { }
```

👉 améliore les performances.

---

# 🗃️ 3. JPA & Hibernate

## ✅ Utiliser DTO au lieu d’exposer les entités

❌ Mauvais :

```java
@GetMapping
public Patient get()
```

✅ Bon :

```java
public PatientDTO get()
```

👉 évite les problèmes de sérialisation et sécurité.

---

## ✅ Charger uniquement ce qui est nécessaire

✔ utiliser projections ou DTO
✔ éviter `FetchType.EAGER`

👉 privilégier :

```java
@ManyToOne(fetch = FetchType.LAZY)
```

---

## ✅ Éviter N+1 queries

Utiliser :

```java
@EntityGraph
JOIN FETCH
```

---

# ⚡ 4. Performance & Scalabilité

## ✅ Pagination obligatoire pour grandes listes

```java
Pageable pageable
```

👉 évite surcharge mémoire.

---

## ✅ Activer le cache si nécessaire

* Cache de second niveau (Hibernate)
* Cache applicatif (Redis)

---

## ✅ Logs SQL uniquement en dev

```properties
spring.jpa.show-sql=true
```

❌ désactiver en production.

---

# 🔐 5. Sécurité & Validation

## ✅ Valider les entrées avec Bean Validation

```java
@NotNull
@Email
@Size(min = 3)
```

👉 empêche données invalides.

---

## ✅ Ne jamais exposer les exceptions techniques

Créer un handler global :

```java
@RestControllerAdvice
```

---

# ⚙️ 6. Configuration & Environnements

## ✅ Utiliser des profils Spring

```properties
spring.profiles.active=dev
```

* dev
* test
* prod

---

## ✅ Externaliser la configuration sensible

❌ éviter mots de passe dans code
✔ utiliser variables d’environnement.

---

# 🧪 7. Tests

## ✅ Écrire plusieurs niveaux de tests

* Unit tests (Mockito)
* Integration tests
* @DataJpaTest
* @WebMvcTest

---

## ✅ Utiliser Testcontainers pour les DB réelles

👉 plus fiable que H2 pour PostgreSQL.

---

# 🚀 8. Gestion des Exceptions

## ✅ Créer des exceptions métier

```java
throw new PatientNotFoundException();
```

---

## ✅ Centraliser la gestion

```java
@ControllerAdvice
```

---

# 🧰 9. Bonnes pratiques REST API

## ✅ Utiliser des URLs cohérentes

✔ `/patients`
✔ `/patients/{id}`

❌ `/getAllPatients`

---

## ✅ Utiliser les bons codes HTTP

| Action      | Code |
| ----------- | ---- |
| Création    | 201  |
| Succès      | 200  |
| Suppression | 204  |
| Non trouvé  | 404  |

---

# 🧠 10. Qualité & Maintenabilité

## ✅ Utiliser Lombok pour réduire le boilerplate

## ✅ Documenter l’API avec Swagger / OpenAPI

## ✅ Utiliser MapStruct pour mapper DTO ↔ Entity

---

# 🎯 Résumé des règles critiques

✔ Architecture en couches
✔ @Transactional dans service
✔ DTO au lieu d’entités
✔ Pagination & Lazy loading
✔ Validation des entrées
✔ Gestion globale des exceptions
✔ Profils & configuration sécurisée
✔ Tests automatisés

---

#  Difference Between `@NotBlank` and `@NotNull`

Both come from Jakarta Bean Validation (used by **Spring Boot**).

But they validate *different things*.

---

##  `@NotNull`

```java
@NotNull
String name;
```

### What it checks:

✔ Value is **not null**

### What it allows:

✔ `""` (empty string)
✔ `"   "` (only spaces)

So this passes:

```java
name = "";
name = "   ";
```

But this fails:

```java
name = null;
```

###  Use it when:

* You only care that the value exists
* It can be empty
* It’s not necessarily a string (works on any object)

---

##  `@NotBlank`

```java
@NotBlank
String name;
```

### What it checks:

✔ Not null
✔ Not empty
✔ Not only whitespace

So this fails:

```java
null
""
"   "
```

And only this passes:

```java
"Zakaria"
```

###  Use it when:

* You validate user text input
* You want real content
* It’s a `String`

---

##  There is also `@NotEmpty` (important)

For comparison:

| Annotation  | Null allowed? | Empty "" allowed? | Only spaces allowed? |
| ----------- | ------------- | ----------------- | -------------------- |
| `@NotNull`  | ❌             | ✅                 | ✅                    |
| `@NotEmpty` | ❌             | ❌                 | ✅                    |
| `@NotBlank` | ❌             | ❌                 | ❌                    |

---

#  Real API Example

For a DTO:

```java
public record PatientDto(

    @NotBlank
    String name,

    @Email
    @NotBlank
    String email
) {}
```

This ensures:

* No null
* No empty
* No whitespace-only input

That’s what you usually want in REST APIs.

---

#  What is `@ControllerAdvice`?

This is where things become professional-level.

`@ControllerAdvice` is a **global exception handler mechanism** in Spring.

Instead of writing try/catch in every controller, you centralize error handling.

---

##  Without `@ControllerAdvice`

You might do this:

```java
@GetMapping("/{id}")
public ResponseEntity<Patient> get(@PathVariable Long id) {
    try {
        return ResponseEntity.ok(service.find(id));
    } catch (Exception e) {
        return ResponseEntity.status(404).build();
    }
}
```

This becomes messy quickly.

---

##  With `@ControllerAdvice`

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<String> handleNotFound(EntityNotFoundException ex) {
        return ResponseEntity
                .status(404)
                .body(ex.getMessage());
    }
}
```

Now:

* Any controller throwing `EntityNotFoundException`
* Automatically returns 404
* No duplication
* Clean architecture

---

#  Why It’s Powerful

It allows you to:

✔ Standardize API error responses
✔ Handle validation errors globally
✔ Log exceptions in one place
✔ Create consistent error JSON format

---

#  Handling Validation Errors Properly

When validation fails (`@Valid`), Spring throws:

```
MethodArgumentNotValidException
```

You can catch it globally:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidationErrors(
            MethodArgumentNotValidException ex) {

        Map<String, String> errors = new HashMap<>();

        ex.getBindingResult().getFieldErrors()
                .forEach(error ->
                        errors.put(error.getField(), error.getDefaultMessage())
                );

        return ResponseEntity.badRequest().body(errors);
    }
}
```

Now your API returns clean structured validation errors.

---

#  Architecture Perspective

In well-designed APIs:

* Controllers = only business flow
* Service = logic
* `@ControllerAdvice` = error strategy
* DTO = validated contract

This separation improves:

* Maintainability
* Scalability
* Readability
* Testing

---

# 🚀 Forward-Thinking Tip

For production-grade systems:

Instead of returning raw strings, create a structured error model:

```java
public record ApiError(
    String message,
    int status,
    LocalDateTime timestamp
) {}
```

Then return consistent JSON for every error.

That’s how mature APIs are built.

The Short Answer
@RestControllerAdvice

is basically:

@ControllerAdvice + @ResponseBody

That’s it.

But let’s go deeper.

@ControllerAdvice

This is a global component that applies to all controllers.

It allows you to:

Handle exceptions globally

Add global model attributes

Bind data across controllers

Example
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(RuntimeException.class)
    public String handleError(Model model, RuntimeException ex) {
        model.addAttribute("errorMessage", ex.getMessage());
        return "error-page";
    }
}

 This returns a view name (like Thymeleaf, JSP, etc.).

So @ControllerAdvice is typically used in MVC applications that render HTML pages.

 @RestControllerAdvice

This is specialized for REST APIs.

It automatically adds @ResponseBody behavior.

So instead of returning a view, it returns JSON.

Example
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(RuntimeException.class)
    public ResponseEntity<String> handleError(RuntimeException ex) {
        return ResponseEntity
                .status(500)
                .body(ex.getMessage());
    }
}

 This returns JSON (or raw body) instead of a view.

 Practical Difference
| Feature                       | @ControllerAdvice       | @RestControllerAdvice |
| ----------------------------- | ----------------------- | --------------------- |
| Used for                      | MVC (HTML views)        | REST APIs             |
| Returns view?                 | ✅ Yes                   | ❌ No                  |
| Returns JSON automatically?   | ❌ Needs `@ResponseBody` | ✅ Yes                 |
| Best for Spring Boot REST app | ❌                       | ✅                     |
🎯 Real-World Rule

If you are building:

🖥 Traditional web app → @ControllerAdvice

🌐 REST API backend → @RestControllerAdvice

In modern Spring Boot APIs, you almost always use:

@RestControllerAdvice
🧠 Why This Matters Architecturally

In REST:

Errors should be JSON

Clients (frontend, mobile, other services) expect structured responses

No server-side view rendering

So @RestControllerAdvice aligns with API-first design.

🚀 Clean API Error Example
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(EntityNotFoundException ex) {
        ApiError error = new ApiError(
                ex.getMessage(),
                404,
                LocalDateTime.now()
        );

        return ResponseEntity.status(404).body(error);
    }
}

Now every error is consistent and structured.

That’s how scalable APIs are built.
---




Voici **tous les codes de statut HTTP officiels** (avec leurs significations) tels que définis par la spécification **RFC 9110** (Internet standard) — regroupés par catégories pour une lecture claire 👇 ([MDN Web Docs][1])



---



## 📘 **1xx — Réponses informationnelles**



| Code    | Signification                                                                                |

| ------- | -------------------------------------------------------------------------------------------- |

| **100** | Continue — le serveur a reçu les en-têtes, peut poursuivre ([MDN Web Docs][1])               |

| **101** | Switching Protocols — changement de protocole (par ex. HTTP → WebSocket) ([MDN Web Docs][1]) |

| **102** | Processing — le serveur traite toujours la requête (WebDAV) ([serverheaders.com][2])         |

| **103** | Early Hints — premières informations avant la réponse finale ([Hostinger][3])                |



---



## ✅ **2xx — Succès (Successful Responses)**



| Code    | Signification                                                                          |

| ------- | -------------------------------------------------------------------------------------- |

| **200** | OK — succès général ([serverheaders.com][2])                                           |

| **201** | Created — ressource créée ([serverheaders.com][2])                                     |

| **202** | Accepted — requête acceptée mais pas encore traitée ([serverheaders.com][2])           |

| **203** | Non-Authoritative Information — réponse modifiée par un proxy ([serverheaders.com][2]) |

| **204** | No Content — requête traitée, sans contenu de réponse ([serverheaders.com][2])         |

| **205** | Reset Content — réinitialiser le document affiché ([serverheaders.com][2])             |

| **206** | Partial Content — contenu partiel (range requests) ([serverheaders.com][2])            |

| **207** | Multi-Status — WebDAV ([serverheaders.com][2])                                         |

| **208** | Already Reported — WebDAV optimisation ([serverheaders.com][2])                        |

| **226** | IM Used — Delta encoding ([serverheaders.com][2])                                      |



---



## 🔁 **3xx — Redirections**



| Code    | Signification                                                                                |

| ------- | -------------------------------------------------------------------------------------------- |

| **300** | Multiple Choices — plusieurs options possibles ([serverheaders.com][2])                      |

| **301** | Moved Permanently — ressource déplacée définitivement ([serverheaders.com][2])               |

| **302** | Found — déplacement temporaire ([serverheaders.com][2])                                      |

| **303** | See Other — voir une autre URI ([serverheaders.com][2])                                      |

| **304** | Not Modified — pas modifié (cache) ([serverheaders.com][2])                                  |

| **305** | Use Proxy — utiliser proxy (rare) ([serverheaders.com][2])                                   |

| **306** | Switch Proxy — désuet ([serverheaders.com][2])                                               |

| **307** | Temporary Redirect — redirection temporaire sans changer la méthode ([serverheaders.com][2]) |

| **308** | Permanent Redirect — redirection permanente sans changer la méthode ([serverheaders.com][2]) |



---



## 🚫 **4xx — Erreurs client**



| Code    | Signification                                                                              |

| ------- | ------------------------------------------------------------------------------------------ |

| **400** | Bad Request — requête invalide ([serverheaders.com][2])                                    |

| **401** | Unauthorized — authentification requise ou incorrecte ([serverheaders.com][2])             |

| **402** | Payment Required — réservé (rare) ([Wikipedia][4])                                         |

| **403** | Forbidden — refus d’accès ([serverheaders.com][2])                                         |

| **404** | Not Found — ressource introuvable ([serverheaders.com][2])                                 |

| **405** | Method Not Allowed — méthode HTTP non autorisée ([serverheaders.com][2])                   |

| **406** | Not Acceptable — contenu non acceptable ([serverheaders.com][2])                           |

| **407** | Proxy Authentication Required — proxy demande auth ([serverheaders.com][2])                |

| **408** | Request Timeout — délai dépassé ([serverheaders.com][2])                                   |

| **409** | Conflict — conflit d’état ([serverheaders.com][2])                                         |

| **410** | Gone — ressource définitivement disparue ([serverheaders.com][2])                          |

| **411** | Length Required — longueur requise ([serverheaders.com][2])                                |

| **412** | Precondition Failed — précondition non satisfaite ([serverheaders.com][2])                 |

| **413** | Payload Too Large — corps trop volumineux ([serverheaders.com][2])                         |

| **414** | URI Too Long — URI trop longue ([serverheaders.com][2])                                    |

| **415** | Unsupported Media Type — type non supporté ([serverheaders.com][2])                        |

| **416** | Range Not Satisfiable — plage non satisfiable ([serverheaders.com][2])                     |

| **417** | Expectation Failed — attente impossible ([serverheaders.com][2])                           |

| **418** | I’m a teapot — erreur humoristique (RFC 2324) ([serverheaders.com][2])                     |

| **421** | Misdirected Request — mauvaise direction ([serverheaders.com][2])                          |

| **422** | Unprocessable Entity — entité non traitable ([serverheaders.com][2])                       |

| **423** | Locked — verrouillé ([serverheaders.com][2])                                               |

| **424** | Failed Dependency — dépendance échouée ([serverheaders.com][2])                            |

| **426** | Upgrade Required — mise à niveau requise ([serverheaders.com][2])                          |

| **428** | Precondition Required — précondition requise ([serverheaders.com][2])                      |

| **429** | Too Many Requests — trop de requêtes ([serverheaders.com][2])                              |

| **431** | Request Header Fields Too Large — en-têtes trop grands ([serverheaders.com][2])            |

| **451** | Unavailable For Legal Reasons — non disponible pour raison légale ([serverheaders.com][2]) |



---



## ⚠ **5xx — Erreurs serveur**



| Code    | Signification                                                                  |

| ------- | ------------------------------------------------------------------------------ |

| **500** | Internal Server Error — erreur serveur interne ([serverheaders.com][2])        |

| **501** | Not Implemented — non implémenté ([serverheaders.com][2])                      |

| **502** | Bad Gateway — mauvaise passerelle ([serverheaders.com][2])                     |

| **503** | Service Unavailable — service indisponible ([serverheaders.com][2])            |

| **504** | Gateway Timeout — délai d’attente passerelle ([serverheaders.com][2])          |

| **505** | HTTP Version Not Supported — version non supportée ([serverheaders.com][2])    |

| **506** | Variant Also Negotiates — négociation alternative ([serverheaders.com][2])     |

| **507** | Insufficient Storage — espace insuffisant ([serverheaders.com][2])             |

| **508** | Loop Detected — boucle détectée ([serverheaders.com][2])                       |

| **510** | Not Extended — extension requise ([serverheaders.com][2])                      |

| **511** | Network Authentication Required — auth réseau requise ([serverheaders.com][2]) |



---



## 📌 Notes importantes



### ⭐ Classes de codes



* **1xx** : informations intermédiaires ([MDN Web Docs][1])

* **2xx** : succès ([MDN Web Docs][1])

* **3xx** : redirections ([MDN Web Docs][1])

* **4xx** : erreur client ([MDN Web Docs][1])

* **5xx** : erreur serveur ([MDN Web Docs][1])



---



## 📌 Bonus (non standards / utilisés par certains serveurs API)



Ceux-ci ne sont **pas dans la norme officielle**, mais on les rencontre parfois :



| Code    | Signification / Usage                                    |

| ------- | -------------------------------------------------------- |

| **420** | Enhance Your Calm (Twitter API rate limit) ([Reddit][5]) |

| **440** | Login Timeout (IIS) ([Wikipedia][6])                     |

| **449** | Retry With (IIS) ([Wikipedia][6])                        |

| **419** | Page Expired (Laravel) ([Wikipedia][6])                  |

---

# 🔷 I) Principes fondamentaux de conception OO

## 1️⃣ SOLID

* **S** — Single Responsibility Principle
* **O** — Open/Closed Principle
* **L** — Liskov Substitution Principle
* **I** — Interface Segregation Principle
* **D** — Dependency Inversion Principle

---

## 2️⃣ GRASP (Responsabilités)

* Information Expert
* Creator
* Controller
* Low Coupling
* High Cohesion
* Polymorphism
* Pure Fabrication
* Indirection
* Protected Variations

---

## 3️⃣ Autres principes OO essentiels

* DRY (Don't Repeat Yourself)
* KISS (Keep It Simple, Stupid)
* YAGNI (You Aren’t Gonna Need It)
* Separation of Concerns
* Encapsulation
* Abstraction
* Composition over Inheritance
* Favor immutability
* Law of Demeter

---

# 🔷 II) Concepts fondamentaux de modélisation UML

Basés sur le standard défini par l’**Object Management Group**.

---

## 1️⃣ Relations UML

* Association
* Agrégation
* Composition
* Généralisation (Héritage)
* Réalisation (Interface)
* Dépendance

---

## 2️⃣ Multiplicité

* 1..1
* 0..1
* 1..*
* 0..*

---

## 3️⃣ Types de diagrammes UML

### Structurels

* Diagramme de classes
* Diagramme d’objets
* Diagramme de composants
* Diagramme de déploiement
* Diagramme de packages

### Comportementaux

* Diagramme de séquence
* Diagramme d’activités
* Diagramme d’états
* Diagramme de cas d’utilisation
* Diagramme de communication

---

# 🔷 III) Cohésion & Couplage

## Haute Cohésion

Une classe fait **une seule chose bien**.

## Faible Couplage

Dépendances minimales entre classes.

---

# 🔷 IV) Anti-patterns à éviter

* God Object
* Anemic Domain Model
* Spaghetti Code
* Circular Dependencies
* Tight Coupling
* Shotgun Surgery
* Feature Envy
* Big Ball of Mud
* Primitive Obsession
* Duplicate Code

---

# 🔷 V) Design Patterns (GoF)

Issus du livre de la **Design Patterns: Elements of Reusable Object-Oriented Software**

---

## 1️⃣ Créationnels

* Singleton
* Factory Method
* Abstract Factory
* Builder
* Prototype

---

## 2️⃣ Structurels

* Adapter
* Bridge
* Composite
* Decorator
* Facade
* Flyweight
* Proxy

---

## 3️⃣ Comportementaux

* Strategy
* Observer
* Command
* State
* Chain of Responsibility
* Mediator
* Memento
* Template Method
* Visitor
* Interpreter

---

# 🔷 VI) Architecture Logicielle

## 1️⃣ Clean Architecture

Concept popularisé par **Robert C. Martin**

* Séparation par couches
* Indépendance des frameworks
* Dépendances vers l’intérieur

---

## 2️⃣ Domain-Driven Design (DDD)

Proposé par **Eric Evans**

Concepts clés :

* Entity
* Value Object
* Aggregate
* Aggregate Root
* Repository
* Domain Service
* Ubiquitous Language
* Bounded Context

---

## 3️⃣ Architectures courantes

* Layered Architecture
* Hexagonal Architecture (Ports & Adapters)
* Onion Architecture
* Microservices
* Event-Driven Architecture
* CQRS
* Event Sourcing

---

# 🔷 VII) Concepts avancés de qualité

## 1️⃣ Métriques de conception

* Complexité cyclomatique
* Instabilité
* Abstractness
* Afferent/Efferent Coupling
* Maintainability Index

---

## 2️⃣ Propriétés de qualité (ISO 25010)

* Maintainability
* Scalability
* Reliability
* Testability
* Security
* Portability
* Performance

---

# 🔷 VIII) Modélisation métier

* Modèle conceptuel
* Modèle logique
* Modèle physique
* Diagramme entité-association
* Normalisation des données
* Cardinalités

---

# 🔷 IX) Bonnes pratiques générales

* Nommer clairement les classes
* Éviter les dépendances circulaires
* Favoriser les interfaces
* Séparer domaine et infrastructure
* Centraliser la gestion d’erreurs
* Utiliser des DTO pour exposer les données
* Versionner les APIs
* Documenter avec OpenAPI

---

# 🔷 X) Concepts liés à la maintenabilité

* Refactoring
* Code Review
* Tests unitaires
* Tests d’intégration
* TDD
* BDD
* CI/CD

---

# 🎯 Résumé Global

La conception et la modélisation reposent sur :

* ✔ Principes fondamentaux (SOLID, GRASP)
* ✔ Bonne gestion des dépendances
* ✔ Modélisation claire des relations
* ✔ Cohérence métier
* ✔ Évolutivité
* ✔ Simplicité
* ✔ Testabilité
* ✔ Séparation des responsabilités

---











