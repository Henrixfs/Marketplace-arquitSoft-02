# Clean Architecture — Diagrama y Tabla

## Diagrama: Las 4 Capas

```mermaid
flowchart TB
    subgraph MOD["MÓDULO (Clean Architecture)"]
        subgraph PRES["PRESENTACIÓN"]
            direction LR
            Routes["📍 routes<br/>GET, POST, PUT"]
            Ctrl["🎛️ controller<br/>Recibe HTTP<br/>Valida entrada"]
            Routes --> Ctrl
        end

        subgraph APP["APLICACIÓN"]
            direction LR
            Service["⚙️ service<br/>Orquesta<br/>dominio + infra"]
            DTO["📦 DTO<br/>Data Transfer"]
            Exc["⚠️ Excepciones"]
            Service --> DTO & Exc
        end

        subgraph DOM["DOMINIO"]
            direction LR
            Entity["👥 entity<br/>Reglas puras"]
            VO["🏷️ value-object"]
            UC["📋 use-case"]
            Entity --> VO & UC
        end

        subgraph INF["INFRAESTRUCTURA"]
            direction LR
            Repo["💾 repository<br/>ORM / SQL"]
            Adapter["🔌 adapter<br/>APIs externas"]
            Repo --> Adapter
        end

        PRES --> APP
        APP --> DOM
        APP --> INF
    end

    DB[("🗄️ PostgreSQL")]
    EXT["🔗 External"]

    INF --> DB & EXT

    classDef pres fill:#e3f2fd,stroke:#1e88e5,stroke-width:2px,color:#000
    classDef app fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#000
    classDef dom fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px,color:#000
    classDef inf fill:#e1bee7,stroke:#6a1b9a,stroke-width:2px,color:#000

    class Routes,Ctrl pres
    class Service,DTO,Exc app
    class Entity,VO,UC dom
    class Repo,Adapter inf

    style PRES fill:#e3f2fd,stroke:#1e88e5,stroke-width:2px
    style APP fill:#fff3e0,stroke:#e65100,stroke-width:2px
    style DOM fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style INF fill:#e1bee7,stroke:#6a1b9a,stroke-width:2px
```

---

## Tabla: Capas y Responsabilidades

| Capa | Archivos | Responsabilidad | No Contiene |
|------|----------|-----------------|------------|
| **PRESENTACIÓN** | `*.routes.js`, `*.controller.js` | Recibir HTTP, validar entrada, responder JSON | Lógica de negocio |
| **APLICACIÓN** | `*.service.js`, `dtos/`, `exceptions/` | Orquestar dominio + infraestructura | Detalles técnicos (Express, ORM) |
| **DOMINIO** | `*.entity.js`, `*.value-object.js`, `*.use-case.js` | Reglas de negocio puras | Express, ORM, Base de datos |
| **INFRAESTRUCTURA** | `*.repository.js`, `*.adapter.js` | Persistencia, APIs externas, caché | Lógica de negocio |
