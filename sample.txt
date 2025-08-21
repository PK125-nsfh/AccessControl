# DIGIT Access Control — Flow, Class & Architecture Diagrams

> Target: eGov DIGIT Core Service for Access Control (ACS)

---

## 1) Authorization Flow (Request → Decision)

```mermaid
flowchart LR
  %% Actors
  U[Client (Citizen/Employee App)] -->|API call with JWT| GW[API Gateway / Ingress (PEP)]
  GW -->|Validate JWT| AS[Auth Service / IdP]
  AS -->|ok/claims| GW

  %% Decision flow via ACS
  GW -->|/authorize {action, tenant, roles}| ACS[Access Control Service (PDP)]
  ACS -->|fetch role-action mappings| MDMS[(MDMS Actions Master)]
  ACS -->|get user roles\n(tenant-aware)| US[User Service]
  ACS -->|optional cache lookup| RC[(Redis Cache)]
  ACS -->|audit decision| K[(Kafka Audit Topic)]
  ACS -->|ALLOW/DENY + allowedActions| GW

  %% Downstream service
  GW -->|forward request if ALLOW| SVC[Domain Microservice]
  GW -->|403 if DENY| U

  %% Tenancy nuance
  subgraph Tenancy
    direction TB
    T1[tenantId of request] --> ACS
    US -->|cross-ULB role resolution| ACS
  end
```

**Notes**

* Actions are API endpoints or UI events defined as MDMS masters; ACS validates if any of the user’s roles (scoped by tenant) map to the requested action.
* Audit of allow/deny can be pushed to Kafka; results are cached for performance.

---

## 2) Core Domain Model (Class Diagram)

```mermaid
classDiagram
  class User {
    +UUID id
    +String username
    +String type  // CITIZEN | EMPLOYEE | SYSTEM
  }

  class Tenant {
    +String tenantId
    +String name
  }

  class Role {
    +String code
    +String name
    +String description
    +boolean isStateLevel
  }

  class Action {
    +String name
    +String url // API path or FE event key
    +String serviceName
    +String method // GET/POST/...
    +String module
    +String feature
  }

  class UserRole {
    +UUID id
    +String userId
    +String roleCode
    +String tenantId
  }

  class RoleAction {
    +UUID id
    +String roleCode
    +String actionName
    +String tenantId
  }

  class AuthorizationRequest {
    +String actionName
    +String tenantId
    +String userId
    +List~String~ userRoles
    +Map~String,String~ attributes // optional ABAC attrs
  }

  class AuthorizationDecision {
    +boolean allowed
    +List~String~ allowedActions
    +String reason
    +long ttlMillis
  }

  User "1" --> "*" UserRole : has
  Tenant "1" --> "*" UserRole : scopes
  Role "1" --> "*" UserRole : assigned via

  Role "1" --> "*" RoleAction : maps
  Action "1" --> "*" RoleAction : mapped to
  Tenant "1" --> "*" RoleAction : scoped by

  AuthorizationRequest --> User
  AuthorizationRequest --> Tenant
  AuthorizationDecision --> Action : allowedActions reference
```

**Notes**

* `RoleAction` is the tenant-scoped mapping that enables cross-ULB authorization (e.g., employee acts as APPROVER in another ULB).
* `AuthorizationDecision.allowedActions` can power FE menu rendering.

---

## 3) High-Level Architecture (Microservices & Data)

```mermaid
graph TB
  subgraph Client Layer
    FE[Web/Mobile Frontend]
  end

  subgraph Edge Layer
    GW[API Gateway / Ingress]
  end

  subgraph Identity & Access
    AS[Auth Service / IdP (JWT/OIDC)]
    ACS[Access Control Service]
  end

  subgraph Core Registries
    US[User Service]
    MDMS[(MDMS)]
  end

  subgraph Data & Infra
    DB[(Postgres/DB for ACS metadata)]
    CACHE[(Redis Cache)]
    K[(Kafka for Audit/Events)]
    OBS[(Observability: Prometheus/Grafana/ELK)]
  end

  subgraph Domain Services
    SVC1[Service A]
    SVC2[Service B]
    SVCn[...]
  end

  FE -->|JWT| GW
  GW --> AS
  GW -->|/authorize| ACS
  ACS --> MDMS
  ACS --> US
  ACS --> CACHE
  ACS --> DB
  ACS --> K
  GW -->|if ALLOW| SVC1
  GW -->|if ALLOW| SVC2
  GW -->|if ALLOW| SVCn
  AS --> GW
  OBS --- ACS
  OBS --- SVC1
  OBS --- GW
```

**Operational Notes**

* **Caching**: cache role→actions and user→roles (tenant-scoped) with TTL; warm on startup.
* **Resilience**: circuit-breakers on MDMS/User Service; fallback to cache for read paths.
* **Multitenancy**: every call is tenant-aware; enforce tenantId validation at gateway and ACS.
* **DevEx**: expose `/_health`, `/metrics`, and a `/playground` (if applicable) for testing.

---

## 4) Sequence (Optional, for a Single API Call)

```mermaid
sequenceDiagram
  autonumber
  participant FE as Frontend
  participant GW as API Gateway (PEP)
  participant AS as Auth Service (IdP)
  participant ACS as Access Control Service (PDP)
  participant US as User Service
  participant MD as MDMS
  participant S as Domain Service

  FE->>GW: Request /tl-service/_search + JWT
  GW->>AS: Validate token
  AS-->>GW: Claims {userId, roles, tenant}
  GW->>ACS: authorize(action="TL_SEARCH", tenant)
  ACS->>US: getUserRoles(userId, tenant)
  US-->>ACS: roles[]
  ACS->>MD: getActions(module, feature)
  MD-->>ACS: actions[]
  ACS-->>GW: ALLOW
  GW->>S: Forward request
  S-->>GW: 200 OK
  GW-->>FE: 200 OK
```

---

## 5) Implementation Pointers

* Keep action definitions in MDMS; maintain `role-action` mappings as tenant-scoped masters.
* Offer APIs:

  * `POST /authorize` — decision on a specific action.
  * `POST /_getActionsForUser` — menu rendering.
  * CRUD for `roles`, `actions`, `role-action` mappings (admin-only).
* Instrument decisions with correlation IDs and push `ALLOW/DENY` to Kafka for audit.
* Provide Postman/Swagger examples in a `/playground` env.

---

### References (for your convenience)

* DIGIT Core: Access Control Services — overview & key functionalities.
* DIGIT Urban docs — MDMS-backed Actions, role→action mapping, tenant-level scoping.
* User Service — role mapping and user management.
