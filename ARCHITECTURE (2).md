# ARCHITECTURE.md — ZakaWise: Personal Finance Management System

---

## Project Overview

**Project Title**: ZakaWise — Personal Finance Management System
**Domain**: FinTech (Financial Technology)
**Problem Statement**: Many people lack financial management discipline, literacy, and skills. Their finances are spread across different accounts making real-time tracking difficult, they overspend without budget awareness, and they make poor financial decisions without guidance. ZakaWise solves these issues in one unified platform where users can plan, track, and improve their finance management.
**Individual Scope**: Single developer, one semester (~14 weeks). Stack: React (frontend), Java Spring Boot (REST API), PostgreSQL (database), Firebase Authentication (identity). Deployed on Render.

---

## C4 Diagram Overview

The C4 model describes software architecture at four increasing levels of detail:

| Level | Focus |
|---|---|
| **Level 1 — System Context** | Who uses ZakaWise and what external systems does it depend on? |
| **Level 2 — Container** | What are the deployable units that make up the system? |
| **Level 3 — Component** | What are the internal components inside the Spring Boot API? |
| **Level 4 — Code** | Class and entity-level detail for the core domain model |

---

## Level 1 — System Context Diagram

> Shows ZakaWise in relation to its users and all external systems it interacts with.

```mermaid
C4Context
  title System Context Diagram — ZakaWise

  Person(user, "Registered User", "A person who wants to track income and expenses, manage monthly budgets, monitor savings goals, and view their financial dashboard.")

  System(zakawise, "ZakaWise", "Personal finance management web application. Allows users to log transactions, manage budgets, track savings goals, and view a financial analytics dashboard. Built with React, Java Spring Boot, PostgreSQL, and Firebase Auth.")

  System_Ext(firebase_auth, "Firebase Authentication", "Google-managed identity platform. Handles user registration and login via email/password and Google Sign-In. Issues signed Firebase ID tokens (JWTs) verified server-side by the Spring Boot backend.")

  Rel(user, zakawise, "Uses via browser", "HTTPS")
  Rel(zakawise, firebase_auth, "Authenticates users via", "HTTPS / Firebase SDK")
```

---

## Level 2 — Container Diagram

> Shows all deployable containers that make up ZakaWise.

```mermaid
C4Container
  title Container Diagram — ZakaWise

  Person(user, "Registered User", "Uses the web app via browser")

  System_Boundary(zw_boundary, "ZakaWise System") {

    Container(web_app, "React Web Application", "React 18, TailwindCSS, Recharts, Firebase JS SDK", "Single Page Application served in the browser. Provides the transaction management UI, budget tracker, savings goals, and financial dashboard with charts. Uses the Firebase JS SDK to authenticate users and obtain ID tokens sent with every API request.")

    Container(api_server, "Spring Boot REST API", "Java 21, Spring Boot 3, Spring Security, Spring Data JPA", "Core backend service hosted on Render. Exposes all RESTful endpoints. Verifies Firebase ID tokens on every request using the Firebase Admin SDK. Implements all business logic: transaction management, budget calculation, savings goal tracking, and dashboard aggregation.")

    Container(db, "PostgreSQL Database", "PostgreSQL 15 (Render Postgres)", "Stores all persistent data: user profiles (keyed by Firebase UID), transactions, budgets, savings goals, and savings contributions.")
  }

  System_Ext(firebase_auth, "Firebase Authentication", "Manages user identity, issues and validates Firebase ID tokens")

  Rel(user, web_app, "Opens in browser", "HTTPS")
  Rel(web_app, firebase_auth, "Signs in user, obtains Firebase ID token", "HTTPS / Firebase JS SDK")
  Rel(web_app, api_server, "API calls with Firebase ID token in Authorization header", "HTTPS / JSON")
  Rel(api_server, firebase_auth, "Verifies Firebase ID tokens via Admin SDK", "HTTPS / Firebase Admin SDK")
  Rel(api_server, db, "Reads and writes all financial data", "TCP / JDBC / JPA")
```

---

## Level 3 — Component Diagram (Spring Boot REST API)

> Zooms into the Spring Boot API container and shows its internal Spring components.

```mermaid
C4Component
  title Component Diagram — ZakaWise Spring Boot REST API

  Container_Boundary(api_boundary, "Spring Boot REST API (Java 21 / Spring Boot 3)") {

    Component(firebase_filter, "FirebaseAuthFilter", "Spring Security OncePerRequestFilter", "Intercepts every incoming HTTP request. Extracts the Firebase Bearer token from the Authorization header, calls FirebaseAuth.verifyIdToken(), and populates the Spring SecurityContext with the user's Firebase UID.")

    Component(user_ctrl, "UserController", "@RestController — /api/users", "Handles user profile creation on first login, profile retrieval, and profile updates. On first login, creates a PostgreSQL profile record keyed to the Firebase UID.")

    Component(transaction_ctrl, "TransactionController", "@RestController — /api/transactions", "Handles CRUD endpoints for income and expense transactions. Supports filtering and searching by date range, category, amount, and type.")

    Component(budget_ctrl, "BudgetController", "@RestController — /api/budgets", "Handles CRUD for monthly budgets per category. Returns current spend progress for each budget and handles monthly reset.")

    Component(goal_ctrl, "SavingsGoalController", "@RestController — /api/goals", "Handles CRUD for savings goals. Returns progress percentage and estimated days remaining to target.")

    Component(dashboard_ctrl, "DashboardController", "@RestController — /api/dashboard", "Returns aggregated monthly summary (total income, total expenses, net balance), the 5 most recent transactions, and 6-month bar chart data.")

    Component(user_service, "UserService", "@Service", "Business logic for profile management. Creates a user profile on first Firebase login. Handles account deletion and cascading removal of all financial data.")

    Component(transaction_service, "TransactionService", "@Service", "Business logic for creating, updating, deleting (soft), and querying transactions. Triggers BudgetService to recalculate progress after any transaction write.")

    Component(budget_service, "BudgetService", "@Service", "Calculates current monthly spend per budget by summing matching expense transactions. Evaluates progress percentage. Resets budgets at the start of each calendar month.")

    Component(goal_service, "SavingsGoalService", "@Service", "Manages savings goals. Calculates progress percentage and estimated completion date. Handles pause, resume, and delete status transitions.")

    Component(dashboard_service, "DashboardService", "@Service", "Aggregates transaction data into total income, total expenses, net balance, 5 recent transactions, and 6-month chart datasets.")

    Component(user_repo, "UserRepository", "Spring Data JPA Repository", "JPA repository for the User entity.")
    Component(transaction_repo, "TransactionRepository", "Spring Data JPA Repository", "JPA repository for the Transaction entity. Includes custom queries for date range filtering, category aggregation, and monthly totals.")
    Component(budget_repo, "BudgetRepository", "Spring Data JPA Repository", "JPA repository for the Budget entity.")
    Component(goal_repo, "SavingsGoalRepository", "Spring Data JPA Repository", "JPA repository for SavingsGoal entity.")
  }

  ContainerDb(db, "PostgreSQL", "Persistent data store")
  System_Ext(firebase, "Firebase Admin SDK", "ID token verification")

  Rel(firebase_filter, firebase, "Verifies ID token via FirebaseAuth.verifyIdToken()")
  Rel(user_ctrl, user_service, "Delegates to")
  Rel(transaction_ctrl, transaction_service, "Delegates to")
  Rel(budget_ctrl, budget_service, "Delegates to")
  Rel(goal_ctrl, goal_service, "Delegates to")
  Rel(dashboard_ctrl, dashboard_service, "Delegates to")

  Rel(transaction_service, transaction_repo, "Reads/writes transactions")
  Rel(transaction_service, budget_service, "Triggers recalculation after write")
  Rel(budget_service, budget_repo, "Reads/writes budgets")
  Rel(budget_service, transaction_repo, "Queries monthly spend totals")
  Rel(goal_service, goal_repo, "Reads/writes goals")
  Rel(dashboard_service, transaction_repo, "Queries aggregated data")
  Rel(user_service, user_repo, "Reads/writes user profiles")

  Rel(user_repo, db, "Queries via JPA")
  Rel(transaction_repo, db, "Queries via JPA")
  Rel(budget_repo, db, "Queries via JPA")
  Rel(goal_repo, db, "Queries via JPA")
```

---

## Level 4 — Code Diagram (JPA Entities & Service Layer)

> Class-level view of the core JPA entities and their Spring service relationships.

```mermaid
classDiagram
  class User {
    +UUID id
    +String firebaseUid
    +String email
    +String displayName
    +String defaultCurrency
    +BigDecimal monthlyIncomeEstimate
    +Boolean notifyAlerts
    +LocalDateTime createdAt
  }

  class Transaction {
    +UUID id
    +UUID userId
    +TransactionType type
    +BigDecimal amount
    +String category
    +String description
    +LocalDate transactionDate
    +PaymentMethod paymentMethod
    +Boolean isDeleted
    +LocalDateTime createdAt
    +LocalDateTime updatedAt
  }

  class Budget {
    +UUID id
    +UUID userId
    +String category
    +BigDecimal monthlyLimit
    +BigDecimal currentSpend
    +String monthYear
    +LocalDateTime createdAt
  }

  class SavingsGoal {
    +UUID id
    +UUID userId
    +String name
    +BigDecimal targetAmount
    +BigDecimal currentAmount
    +LocalDate targetDate
    +GoalStatus status
    +LocalDateTime createdAt
  }

  class TransactionService {
    +createTransaction(uid, dto) Transaction
    +getTransactions(uid, filters) Page~Transaction~
    +updateTransaction(id, dto) Transaction
    +deleteTransaction(id) void
    -notifyBudgetService(uid, transaction) void
  }

  class BudgetService {
    +createBudget(uid, dto) Budget
    +recalculateProgress(uid, category) void
    +resetMonthlyBudgets() void
    -getCurrentMonthSpend(uid, category) BigDecimal
  }

  class SavingsGoalService {
    +createGoal(uid, dto) SavingsGoal
    +updateGoalStatus(id, status) SavingsGoal
    +getProgress(id) GoalProgressDTO
    -calculateEstimatedDays(goal) int
  }

  class DashboardService {
    +getDashboardSummary(uid) DashboardDTO
    -getMonthlyTotals(uid) TotalsDTO
    -getRecentTransactions(uid) List~Transaction~
    -getSixMonthChartData(uid) List~MonthlyBarDTO~
  }

  TransactionService --> BudgetService : triggers recalculation
  BudgetService --> Transaction : queries monthly spend
  BudgetService --> Budget : reads and updates
  DashboardService --> Transaction : queries recent + aggregates
  User "1" --> "0..*" Transaction : owns
  User "1" --> "0..*" Budget : owns
  User "1" --> "0..*" SavingsGoal : owns
```

---

## End-to-End Request Flow — Add a Transaction

> Shows the complete request lifecycle from the React UI through Firebase token verification to PostgreSQL and back.

```mermaid
sequenceDiagram
  actor User
  participant Browser as React Web App
  participant Firebase as Firebase Auth
  participant API as Spring Boot API
  participant Filter as FirebaseAuthFilter
  participant TS as TransactionService
  participant BS as BudgetService
  participant DB as PostgreSQL

  User->>Browser: Fills transaction form, clicks Save
  Browser->>Firebase: getIdToken() — refreshes if expired
  Firebase-->>Browser: Returns valid Firebase ID token
  Browser->>API: POST /api/transactions\nAuthorization: Bearer <Firebase ID Token>
  API->>Filter: Intercepts request
  Filter->>Firebase: FirebaseAuth.verifyIdToken(token)
  Firebase-->>Filter: DecodedToken (uid, email)
  Filter-->>API: SecurityContext set with uid
  API->>TS: createTransaction(uid, dto)
  TS->>DB: transactionRepository.save(transaction)
  DB-->>TS: Persisted Transaction
  TS->>BS: recalculateProgress(uid, category)
  BS->>DB: Sum expense transactions for category + month
  DB-->>BS: Current spend total
  BS->>DB: budgetRepository.save(updatedBudget)
  API-->>Browser: 201 Created — TransactionResponseDTO
  Browser-->>User: Success toast; dashboard refreshes
```

---

## Deployment Architecture

> ZakaWise is fully deployed on Render — no AWS or Docker required.

```mermaid
graph TB
  subgraph Client["Client (Browser)"]
    B[React SPA]
  end

  subgraph Firebase_Cloud["Google Firebase (Managed)"]
    FA[Firebase Authentication]
  end

  subgraph Render_Cloud["Render Cloud Hosting"]
    subgraph Static["Render Static Site"]
      N[React App — CDN served]
    end
    subgraph Web["Render Web Service"]
      A[Spring Boot API — JVM]
    end
    subgraph DB_Service["Render PostgreSQL"]
      P[(PostgreSQL 15)]
    end
  end

  B -->|HTTPS| N
  B -->|Firebase JS SDK| FA
  N -->|API calls| A
  A -->|Firebase Admin SDK| FA
  A -->|JDBC / JPA| P
```

---

## Technology Mapping Summary

| Concern | Solution |
|---|---|
| User Identity & Login | Firebase Authentication (email/password + Google Sign-In) |
| Frontend | React 18 SPA — TailwindCSS, Recharts for charts |
| API Layer | Java 21 + Spring Boot 3 REST Controllers |
| Token Verification | `FirebaseAuthFilter` (Spring Security) + Firebase Admin SDK `verifyIdToken()` |
| Data Persistence | Spring Data JPA (Hibernate) + PostgreSQL 15 |
| Budget Progress | `BudgetService` — sums matching expense transactions per category per month |
| Dashboard Aggregation | `DashboardService` — queries totals, recent 5 transactions, 6-month chart data |
| Deployment | Render (Static Site for React, Web Service for Spring Boot, Managed PostgreSQL) |
