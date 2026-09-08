# 🏛️ System Architecture Specification (Deus Drones ERP)
> **Standard: IEEE 1016-2009 Systems Design Description (SDD)**

---

## 1. Architectural Goals & Design Philosophy

The **Deus Drones ERP** architecture is designed around the principles of deterministic state management, transactional integrity, and operational simplicity. High-throughput assembly and repair depots require immediate real-time response times with zero risk of concurrency-induced stock anomalies (e.g., negative stock, race conditions during part allocation).

### Core Architectural Principles:
1. **Modular Monolith:** Co-locating presentation, domain logic, and persistence interfaces within a single cohesive deployment unit to eliminate network partitioning overhead while maintaining clean internal boundaries.
2. **Strict Referential Integrity:** Foreign keys with explicit cascading and deletion rules (`ON DELETE RESTRICT` for assembled assets and consumed components) guarantee that historical production records can never be orphaned or corrupted.
3. **Database-Level Concurrency Isolation:** Critical inventory decrementing and state transitions utilize row-level pessimistic locking (`SELECT ... FOR UPDATE`) in PostgreSQL, preventing "lost update" anomalies during simultaneous multi-terminal operations.
4. **Client-Agnostic REST Contracts:** Clear decoupling between the backend API and the presentation layer, allowing standard web browsers, mobile PWA clients, and automated inspection stations to consume the same secure endpoints.

---

## 2. Layered System Decomposition

The system is structured into four distinct, loosely coupled architectural tiers:

```mermaid
graph TD
    subgraph Presentation [Client / Presentation Tier]
        PWA["Responsive PWA Shell (Vanilla JS / CSS Grid)"]
        SW["Service Worker (Offline Shell & Asset Cache)"]
        Scanner["Camera Barcode Scanner (Html5-QRCode)"]
    end

    subgraph Transport [Transport & Security Tier]
        FastAPI["FastAPI REST Router (/api/v1)"]
        OAuth["JWT Authentication Middleware"]
        RBAC["Role-Based Access Control Dependency Injection"]
        AuditMW["Audit Logging Interceptor"]
    end

    subgraph Domain [Domain & Service Tier]
        ItemService["Item & Inventory Ledger Service"]
        BOMService["BOM Assembly Orchestration Engine"]
        CustodyService["Custodial Movement Service (Issue/Return)"]
        RepairService["Maintenance & Overhaul Service"]
        ExportService["Document & Spreadsheet Generation Service"]
    end

    subgraph Persistence [Data Persistence Tier]
        UnitOfWork["SQLAlchemy 2.0 Session / Unit of Work"]
        Locking["Row Lock Manager (SELECT ... FOR UPDATE)"]
        PostgreSQL[("PostgreSQL 16 Relational Engine (11 Normalized Tables)")]
    end

    PWA --> SW
    Scanner --> PWA
    SW --> FastAPI
    FastAPI --> OAuth
    OAuth --> RBAC
    RBAC --> AuditMW
    AuditMW --> ItemService
    AuditMW --> BOMService
    AuditMW --> CustodyService
    AuditMW --> RepairService
    AuditMW --> ExportService
    ItemService --> UnitOfWork
    BOMService --> UnitOfWork
    CustodyService --> UnitOfWork
    RepairService --> UnitOfWork
    ExportService --> UnitOfWork
    UnitOfWork --> Locking
    Locking --> PostgreSQL
```

---

## 3. Relational Schema & Entity-Relationship Model

The PostgreSQL database encompasses **11 relational tables** enforcing strict relational constraints:

```mermaid
erDiagram
    User ||--o{ Item : "responsible_person"
    User ||--o{ IssueReturn : "issued_by"
    User ||--o{ AuditLog : "performed_by"
    User ||--o{ AssemblyLog : "assembled_by"

    Item ||--o{ IssueReturn : "item_id"
    Item ||--o{ Repair : "item_id"
    Item ||--o{ WriteOff : "item_id"
    Item ||--o{ Purchase : "item_id"
    Item ||--o{ AssemblyConsumedItem : "consumed_item_id"

    BOMTemplate ||--o{ BOMItem : "template_id"
    BOMTemplate ||--o{ AssemblyLog : "template_id"

    AssemblyLog ||--|{ AssemblyConsumedItem : "assembly_log_id (RESTRICT)"
    AssemblyLog ||--|| Item : "created_item_id (RESTRICT)"

    User {
        int id PK
        string username UK
        string full_name
        enum role "admin, leader, worker"
        string password_hash
        boolean is_active
        datetime created_at
    }

    Item {
        int id PK
        string inventory_number UK
        string category
        enum accounting_type "PIECE, BULK"
        string name
        string model
        decimal quantity
        string unit_of_measure
        enum readiness_class "G0, G1, G2, G3"
        enum completeness "COMPLETE, INCOMPLETE"
        enum status "OPERATIONAL, INCOMPLETE, IN_REPAIR, DONOR, NEW, WRITTEN_OFF"
        string storage_location
        string serial_number
        string qr_token UK
        decimal min_quantity
        datetime created_at
        datetime updated_at
    }

    IssueReturn {
        int id PK
        int item_id FK
        decimal quantity
        string recipient_person
        date planned_return_date
        date actual_return_date
        string status "ACTIVE, RETURNED"
        datetime issued_at
        datetime returned_at
    }

    Repair {
        int id PK
        int item_id FK
        text failure_description
        decimal cost
        enum repair_status "IN_PROGRESS, COMPLETED, CANCELLED"
        enum class_after_repair "G0, G1, G2, G3"
        datetime created_at
        datetime completed_at
    }

    WriteOff {
        int id PK
        int item_id FK
        decimal quantity
        text reason
        string act_number
        datetime written_off_at
    }

    Purchase {
        int id PK
        int item_id FK
        decimal quantity
        decimal unit_cost
        string supplier
        string invoice_number
        datetime purchased_at
    }

    BOMTemplate {
        int id PK
        string name UK
        string target_category
        text description
        boolean is_active
        datetime created_at
    }

    BOMItem {
        int id PK
        int template_id FK
        string component_category
        decimal default_quantity
        string component_name_pattern
        boolean is_mandatory
    }

    AssemblyLog {
        int id PK
        int template_id FK
        int created_item_id FK "RESTRICT"
        int user_id FK
        string serial_number
        text notes
        datetime assembled_at
    }

    AssemblyConsumedItem {
        int id PK
        int assembly_log_id FK "RESTRICT"
        int item_id FK "RESTRICT"
        decimal quantity_consumed
    }

    AuditLog {
        int id PK
        int user_id FK
        string action
        string target_entity
        string target_id
        jsonb old_state
        jsonb new_state
        string client_ip
        string user_agent
        datetime created_at
    }
```

---

## 4. Concurrency Control & State Consistency

In a high-intensity assembly workshop, race conditions present significant risk. Examples include:
- Two technicians attempting to allocate the same flight controller simultaneously.
- Concurrent issuance and decommissioning of the same hardware unit.
- Simultaneous bulk stock deductions exceeding actual shelf balance.

To guarantee zero negative stock and eliminate "lost update" anomalies, the system implements **Pessimistic Row-Level Locking** via SQLAlchemy `with_for_update()` on PostgreSQL:

```python
# Atomic row locking pattern implemented across all mutating endpoints
with session.begin():
    item = (
        session.query(Item)
        .filter(Item.id == item_id)
        .with_for_update()  # Acquires PostgreSQL row-level lock (FOR UPDATE)
        .first()
    )
    if not item:
        raise HTTPException(status_code=404, detail="Entity not found")
        
    if item.accounting_type == AccountingTypeEnum.BULK:
        if item.quantity < requested_decrement:
            raise HTTPException(status_code=400, detail="Insufficient quantity")
        item.quantity -= requested_decrement
    else:
        # Atomic status mutation for piece-wise serialized assets
        item.status = TargetStatusEnum.IN_REPAIR
```

### Locking Guarantees:
- Transactions acquire locks on specific rows in strict order of entity access.
- Conflicting transactions wait until the locking transaction either commits or rolls back.
- Session boundaries (`with session.begin():`) ensure automatic rollback upon unexpected exceptions, guaranteeing database consistency.

---

## 5. Bill of Materials (BOM) Assembly Execution Workflow

The assembly workflow coordinates sub-component deduction, airframe instantiation, and immutable audit linkage in a single ACID transaction:

```mermaid
sequenceDiagram
    autonumber
    actor Technician as Lead Technician
    participant Client as PWA Web Interface
    participant API as Assembly REST Router
    participant Service as BOM Orchestrator
    participant DB as PostgreSQL 16 (ACID)

    Technician->>Client: Selects BOM Template (e.g., Tactical UAV-7) & Components
    Client->>API: POST /api/v1/assembly/build (TemplateID, Components, NewAirframeData)
    API->>Service: execute_assembly(payload, current_user)
    
    critical Atomic Assembly Transaction
        Service->>DB: BEGIN TRANSACTION
        Service->>DB: SELECT * FROM items WHERE id IN (...) FOR UPDATE
        Note over Service,DB: Locks sub-components to prevent double allocation
        
        Service->>DB: Validate stock availability & status (Must be G0 / Operational)
        
        loop For each consumed component
            alt Bulk Component (e.g., Motors, Props)
                Service->>DB: UPDATE items SET quantity = quantity - consumed_qty
            else Serialized Component (e.g., Flight Controller)
                Service->>DB: UPDATE items SET status = 'CONSUMED', quantity = 0
            end
        end
        
        Service->>DB: INSERT INTO items (New Airframe: DRN-XXXX, status='OPERATIONAL', class='G0')
        Service->>DB: INSERT INTO assembly_logs (TemplateID, CreatedAirframeID, UserID)
        Service->>DB: INSERT INTO assembly_consumed_items (LogID, ItemID, Qty)
        Service->>DB: INSERT INTO audit_logs (Action='ASSEMBLY_COMPLETE', ActorID=UserID)
        Service->>DB: COMMIT TRANSACTION
    end

    DB-->>Service: Transaction Succeeded
    Service-->>API: Created Asset DTO (DRN-XXXX, Tokenized QR)
    API-->>Client: HTTP 201 Created (Airframe Details & Digital Passport)
    Client-->>Technician: Displays Assembled Unit with QR Code
```

---

## 6. Environmental Independence & Deployment Architecture

1. **Path Portability:** Zero absolute filesystem paths exist within the application source code. Dynamic directory derivation uses Python standard `pathlib.Path(__file__).resolve().parent`.
2. **Stateless App Architecture:** The FastAPI backend does not store session state in server memory; authentication is maintained via cryptographically verified JSON Web Tokens (JWT).
3. **Database Portability:** Standard connection strings configured via `.env` (`DATABASE_URL=postgresql://user:pass@host:5432/dbname`), compatible with local portable instances, containerized Docker clusters, or cloud-managed databases (AWS RDS, GCP Cloud SQL).

---

## 7. Related Technical Specifications

- 📄 **[System Landing & Project Overview](README.md)**
- 🔄 **[Traceability & Hardware Lifecycle](TRACEABILITY_AND_LIFECYCLE.md)**
- 🛡️ **[Security, Governance & Resilience](SECURITY_AND_RELIABILITY.md)**
