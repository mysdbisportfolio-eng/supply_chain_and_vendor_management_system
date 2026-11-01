# 07. SYSTEM ARCHITECTURE & COMPONENT TOPOLOGY
**System:** Enterprise Procure-to-Pay (P2P) Supply Chain Modular Monolith  
**Namespace:** `ro.uaic.feaa.sdbis.supplychain`

---

## 1. COMPOSITE ARCHITECTURAL PATTERN

The system synthesizes three architectural paradigms:
1. **Modular Monolith (Package-by-Feature / Bounded Context):** Isolates modules inside a single deliverable (`.jar` or `.war`), preventing premature distributed system complexity.
2. **Clean Architecture (Hexagonal / Ports & Adapters):** Core domain rules occupy the center; persistence, web controllers, and third-party APIs reside on the periphery as adapters.
3. **Event-Driven Architecture (EDA):** Cross-module workflows are coordinated asynchronously via transactional domain events.

```mermaid
graph TD
    subgraph Presentation["Presentation Layer (Periphery)"]
        C1["ComandaCtrl"]
        C2["CalitateCtrl"]
        C3["FacturaCtrl"]
    end

    subgraph Application["Application Layer (Use Case Coordination)"]
        A1["ComandaApplicationService"]
        A2["CalitateApplicationService"]
        A3["FacturaApplicationService"]
        Q1["Query Services"]
    end

    subgraph Domain["Domain Layer (Core Business Entities & Invariants)"]
        D1["ComandaAprovizionare"]
        D2["Furnizor"]
        D3["InspectieCalitate"]
        D4["FacturaFurnizor"]
        D5["OrdinPlata"]
        D6["Domain Services & Policies"]
    end

    subgraph Infrastructure["Infrastructure Layer (Adapters & Persistence)"]
        I1["Spring Data JPA Repositories"]
        I2["StripeBancaPlataAdapter"]
        I3["AmqpEventPublisher"]
        I4["Flyway Migrations V1-V6"]
        I5["OutboxRepoJPA / InboxRepoJPA"]
    end

    C1 --> A1
    C2 --> A2
    C3 --> A3

    A1 --> D1
    A2 --> D2
    A2 --> D3
    A3 --> D4
    A3 --> D5

    A1 -.-> I1
    A2 -.-> I1
    A3 -.-> I1
    A3 -.-> I2
    A1 -.-> I5

    I5 --> I3
```

---

## 2. COMPONENT TOPOLOGY PER BOUNDED CONTEXT

### A. Comenzi Bounded Context Component Model
```mermaid
graph TD
    subgraph ComenziModule["comenzi Module Boundary"]
        CCtrl["ComandaCtrl (REST API)"] --> CCmd["CreeazaComandaCommand"]
        CCtrl --> AppSvc["ComandaApplicationService"]
        AppSvc --> DomainModel["ComandaAprovizionare (Aggregate)"]
        DomainModel --> LineModel["LinieComanda (Entity)"]
        DomainModel --> LotModel["Lot (Entity)"]
        AppSvc --> Repo["ComandaRepoJPA"]
        AppSvc --> Estimator["EstimareLeadTimeService"]
        AppSvc --> PubApi["ComandaModuleFacade -> IComenziModuleApi"]
    end
```

### B. Calitate Bounded Context Component Model
```mermaid
graph TD
    subgraph CalitateModule["calitate Module Boundary"]
        QCtrl["CalitateCtrl (REST API)"] --> QAppSvc["CalitateApplicationService"]
        QAppSvc --> VendorModel["Furnizor (Aggregate)"]
        QAppSvc --> InspModel["InspectieCalitate (Aggregate)"]
        QAppSvc --> NcrModel["RaportNeconformitate (Aggregate)"]
        QAppSvc --> ScoreModel["ScorecardLunar (Aggregate)"]
        QAppSvc --> AQL["PoliticaSelectieAQL (Policy)"]
        QAppSvc --> ScoreCalc["CalculScorecardService (Domain Service)"]
        QAppSvc --> QRepo["FurnizorRepoJPA / InspectieRepoJPA"]
        QAppSvc --> QFacade["CalitateModuleFacade -> ICalitateModuleApi"]
    end
```

### C. Facturi Bounded Context Component Model
```mermaid
graph TD
    subgraph FacturiModule["facturi Module Boundary"]
        FCtrl["FacturaCtrl (REST API & Stripe Webhook)"] --> FAppSvc["FacturaApplicationService"]
        FAppSvc --> FactModel["FacturaFurnizor (Aggregate)"]
        FAppSvc --> NirModel["NIR (Aggregate)"]
        FAppSvc --> OrdinModel["OrdinPlata (Aggregate)"]
        FAppSvc --> Reconcil["ReconciliereService (Domain Service)"]
        FAppSvc --> FRepo["FacturaRepoJPA / NIRRepoJPA"]
        FAppSvc --> Stripe["StripeBancaPlataAdapter -> IBancaPlataProvider"]
        FAppSvc --> FFacade["FacturaModuleFacade -> IFacturiModuleApi"]
    end
```

---

## 3. END-TO-END DATA FLOW ARCHITECTURE

```mermaid
sequenceDiagram
    autonumber
    participant Client as HTTP Client
    participant Web as Presentation Layer (FacturaCtrl)
    participant Sec as RBAC & Audit Filter
    participant App as Application Service
    participant Dom as Domain Aggregate (OrdinPlata)
    participant DB as PostgreSQL Transaction
    participant Out as shared_outbox
    participant Sched as OutboxDispatcherService
    participant RMQ as RabbitMQ Topic Exchange

    Client->>Web: POST /api/ordine-plata/webhook-banca
    Web->>Sec: Validate Stripe-Signature & Replay Window (300s)
    Sec-->>Web: Signature Verified OK
    Web->>App: inregistreazaExecutiePlata(ordinId)
    App->>Dom: ordin.inregistreazaExecutie()
    Dom->>Dom: Set state = Executat and emit PlataExecutataEvent
    App->>DB: UPDATE facturi_ordine_plata SET stare_ordin='Executat'
    App->>Out: INSERT INTO shared_outbox (PlataExecutataPublicEvent)
    Note over DB,Out: Single ACID Commit (No Dual-Write Risk)
    DB-->>App: Commit OK
    App-->>Web: Execution Finalized
    Web-->>Client: HTTP 200 OK

    Note over Sched: Asynchronous Outbox Poller (every 5s)
    Sched->>Out: SELECT * FROM shared_outbox WHERE published=false
    Sched->>RMQ: basicPublish(exchange, "facturi.plata.executata", payload)
    RMQ-->>Sched: ACK
    Sched->>Out: UPDATE shared_outbox SET published=true
```

