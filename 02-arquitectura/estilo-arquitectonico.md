# Diagramas de Arquitectura — Monolito Modular en Capas

---

## DIAGRAMA 1: Arquitectura del Monolito (Visión Global)

Muestra la **estructura global del sistema**: actores, cliente web, monolito con sus 6 módulos, middlewares, adaptadores, bases de datos y sistemas externos.

```mermaid
flowchart TB
    subgraph ACT["ACTORES"]
        direction LR
        Cliente["👤 Cliente"]
        Seller["👤 Seller"]
        Admin["👤 Administrador"]
    end

    Web["🌐 Cliente Web - SPA<br/>HTML · CSS · JavaScript<br/>Navegador"]

    subgraph MONO["« MONOLITO » Marketplace Backend<br/>Node.js LTS + Express · Un proceso · Un despliegue"]
        MW["Middlewares Transversales<br/>cors · express.json() · auth JWT · RBAC<br/>validación de entrada · rate limiting<br/>manejo de errores · logger"]

        subgraph MU["módulo: USUARIOS"]
            direction TB
            UR["usuarios.routes.js"]
            UC["usuarios.controller.js"]
            US["usuarios.service.js<br/>registro · login · roles"]
            UP["usuarios.repository.js"]
            UR --> UC --> US --> UP
        end

        subgraph MS["módulo: SELLERS"]
            direction TB
            SR["sellers.routes.js"]
            SC["sellers.controller.js"]
            SS["sellers.service.js<br/>alta · baja · validación"]
            SP["sellers.repository.js"]
            SR --> SC --> SS --> SP
        end

        subgraph MC["módulo: CATALOGO"]
            direction TB
            CR["catalogo.routes.js"]
            CC["catalogo.controller.js"]
            CS["catalogo.service.js<br/>productos · categorías · stock"]
            CP["catalogo.repository.js"]
            CR --> CC --> CS --> CP
        end

        subgraph MK["módulo: CARRITO"]
            direction TB
            KR["carrito.routes.js"]
            KC["carrito.controller.js"]
            KS["carrito.service.js<br/>ítems · totales"]
            KP["carrito.repository.js"]
            KR --> KC --> KS --> KP
        end

        subgraph MP["módulo: PEDIDOS"]
            direction TB
            PR["pedidos.routes.js"]
            PC["pedidos.controller.js"]
            PS["pedidos.service.js<br/>checkout · estados · coordinación"]
            PP["pedidos.repository.js"]
            PR --> PC --> PS --> PP
        end

        subgraph MG["módulo: PAGOS"]
            direction TB
            GR["pagos.routes.js"]
            GC["pagos.controller.js"]
            GS["pagos.service.js<br/>cobro · webhooks"]
            GP["pagos.repository.js"]
            GR --> GC --> GS --> GP
        end

        subgraph INT["Adaptadores de Integración · src/shared/integrations"]
            direction LR
            AP["pasarela-pago.adapter.js"]
            AE["envio.adapter.js"]
            AF["facturacion.adapter.js"]
            AR["erp.adapter.js"]
        end

        DAL["Acceso a Datos Compartido<br/>ORM · modelos · pool de conexiones"]
    end

    PG[("🗄️ PostgreSQL<br/>marketplace_db")]
    CACHE[("⚡ Redis<br/>caché")]

    EPAGO["🔗 Pasarela de pagos<br/>Culqi / Mercado Pago"]
    EENV["🚚 Servicio de envíos<br/>Courier API"]
    EFACT["📄 Servicio de Facturación<br/>Boletas / Facturas"]
    EERP["📊 ERP<br/>Sincronización de stock"]

    ACT --> Web
    Web -->|"HTTPS<br/>JSON<br>/api/v1/*"| MW
    MW --> UR & SR & CR & KR & PR & GR

    PS -.->|"obtiene"| KS
    PS -.->|"reserva"| CS
    PS -.->|"solicita"| GS
    CS -.->|"valida"| SS

    UP & SP & CP & KP & PP & GP --> DAL
    DAL -->|"SQL · TCP 5432"| PG
    CS -->|"consultas frecuentes"| CACHE

    GS --> AP -->|"HTTPS · REST"| EPAGO
    PS --> AE -->|"HTTPS · REST"| EENV
    PS --> AF -->|"HTTPS · REST"| EFACT
    CS --> AR -->|"HTTPS · REST"| EERP

    classDef actor fill:#fff9c4,stroke:#f9a825,stroke-width:2px,color:#000
    classDef web fill:#e1f5fe,stroke:#0277bd,stroke-width:2px,color:#000
    classDef mw fill:#bbdefb,stroke:#1565c0,stroke-width:2px,color:#000
    classDef pres fill:#e3f2fd,stroke:#1e88e5,color:#000
    classDef neg fill:#c8e6c9,stroke:#2e7d32,color:#000
    classDef dat fill:#ffe0b2,stroke:#ef6c00,color:#000
    classDef adapt fill:#e1bee7,stroke:#6a1b9a,color:#000
    classDef store fill:#fff3e0,stroke:#e65100,color:#000
    classDef ext fill:#eeeeee,stroke:#616161,color:#000

    class Cliente,Seller,Admin actor
    class Web web
    class MW mw
    class UR,UC,SR,SC,CR,CC,KR,KC,PR,PC,GR,GC pres
    class US,SS,CS,KS,PS,GS neg
    class UP,SP,CP,KP,PP,GP,DAL dat
    class AP,AE,AF,AR adapt
    class PG,CACHE store
    class EPAGO,EENV,EFACT,EERP ext

    style MONO fill:#fafafa,stroke:#424242,stroke-width:3px,color:#000
    style ACT fill:#fffde7,stroke:#f9a825,stroke-width:2px,color:#000
    style INT fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px,color:#000
    style MU fill:#ffffff,stroke:#9e9e9e,stroke-dasharray: 5 5,color:#000
    style MS fill:#ffffff,stroke:#9e9e9e,stroke-dasharray: 5 5,color:#000
    style MC fill:#ffffff,stroke:#9e9e9e,stroke-dasharray: 5 5,color:#000
    style MK fill:#ffffff,stroke:#9e9e9e,stroke-dasharray: 5 5,color:#000
    style MP fill:#ffffff,stroke:#9e9e9e,stroke-dasharray: 5 5,color:#000
    style MG fill:#ffffff,stroke:#9e9e9e,stroke-dasharray: 5 5,color:#000
```

## DIAGRAMA 2: Arquitectura en Capas (Visión Interna)

Muestra cómo está **organizado cada módulo internamente** en capas, siguiendo Clean Architecture.

```mermaid
flowchart TB
    subgraph MOD["MÓDULO (ej: Usuarios)"]
        subgraph PRES["CAPA 1: PRESENTACIÓN"]
            direction LR
            Routes["📍 usuarios.routes.js<br/>GET, POST, PUT, DELETE"]
            Ctrl["🎛️ usuarios.controller.js<br/>Recibe HTTP<br/>Valida entrada<br/>Responde JSON"]
            Routes --> Ctrl
        end

        subgraph APP["CAPA 2: APLICACIÓN"]
            direction LR
            Service["⚙️ UsuariosService<br/>Orquesta dominio<br/>Coordina con infra"]
            DTO["📦 UsuarioDTO<br/>Data Transfer Object"]
            Exc["⚠️ Excepciones<br/>UsuarioNoEncontrado<br/>EmailYaExiste"]
            Service --> DTO & Exc
        end

        subgraph DOM["CAPA 3: DOMINIO"]
            direction LR
            Entity["👥 Usuario (Entity)<br/>id, email, rol"]
            Rules["📋 Reglas de Negocio<br/>esEmailValido()<br/>puedeCrearPedido()"]
            VO["🏷️ Value Objects<br/>Email, Contraseña<br/>Precio"]
            Entity --> Rules & VO
        end

        subgraph INF["CAPA 4: INFRAESTRUCTURA"]
            direction LR
            Repo["💾 UsuarioRepository<br/>Implementa persistencia<br/>Usa ORM / SQL"]
            Adapter["🔌 Notificadores<br/>EmailNotifier<br/>SMSNotifier"]
            Client["🌐 ClientesHTTP<br/>Para APIs externas"]
            Repo --> Adapter & Client
        end

        PRES --> APP
        APP --> DOM
        APP --> INF
    end

    DB[("🗄️ PostgreSQL<br/>Tablas: usuarios")]
    EXT["🔗 Servicios Externos<br/>Email, SMS, etc"]

    INF -->|"SQL"| DB
    INF -->|"HTTPS"| EXT

    classDef pres fill:#e3f2fd,stroke:#1e88e5,stroke-width:2px,color:#000
    classDef app fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#000
    classDef dom fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px,color:#000
    classDef inf fill:#e1bee7,stroke:#6a1b9a,stroke-width:2px,color:#000
    classDef data fill:#ffccbc,stroke:#d84315,stroke-width:2px,color:#000

    class Routes,Ctrl pres
    class Service,DTO,Exc app
    class Entity,Rules,VO dom
    class Repo,Adapter,Client inf
    class DB,EXT data

    style PRES fill:#e3f2fd,stroke:#1e88e5,stroke-width:2px
    style APP fill:#fff3e0,stroke:#e65100,stroke-width:2px
    style DOM fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style INF fill:#e1bee7,stroke:#6a1b9a,stroke-width:2px
    style MOD fill:#fafafa,stroke:#424242,stroke-width:2px
```

### Explicación de las Capas

| Capa | Responsabilidad | No contiene | Ejemplo |
|------|-----------------|-------------|---------|
| **PRESENTACIÓN** | Recibir HTTP, validar entrada, responder JSON | Lógica de negocio | `POST /api/v1/usuarios/registro` |
| **APLICACIÓN** | Orquestar dominio + infraestructura, casos de uso | Detalles técnicos | `RegistrarUsuarioService.registrar()` |
| **DOMINIO** | Reglas de negocio puras, independiente | Express, ORM, BD | `Usuario.esEmailValido()` |
| **INFRAESTRUCTURA** | Detalles técnicos: ORM, APIs, caché | Lógica de negocio | `UsuarioRepository.guardar(usuario)` |

---

## DIAGRAMA 3: Flujo de Ejecución (Caso de Uso: Crear Pedido)

Muestra cómo interactúan las capas y los módulos durante el checkout (RF05).

```mermaid
sequenceDiagram
    autonumber
    actor Cliente
    participant WEB as Cliente Web
    participant MW as Middlewares
    participant PED_CTRL as pedidos.controller
    participant PED_SVC as pedidos.service
    participant PED_DOM as Pedido (domain)
    participant CAR_SVC as carrito.service
    participant CAT_SVC as catalogo.service
    participant PAG_SVC as pagos.service
    participant PED_REPO as pedidos.repository
    participant DB as PostgreSQL

    Cliente->>WEB: Click "Confirmar compra"
    WEB->>MW: POST /api/v1/pedidos<br/>(JWT, tokenPago)
    MW->>PED_CTRL: Usuario autenticado ✓
    
    PED_CTRL->>PED_SVC: crearPedido(clienteId, monto)
    
    PED_SVC->>CAR_SVC: obtenerCarrito(clienteId)
    CAR_SVC-->>PED_SVC: {ítems, total}
    
    PED_SVC->>CAT_SVC: reservarStock(ítems)
    CAT_SVC-->>PED_SVC: Stock reservado ✓
    
    PED_SVC->>PAG_SVC: procesar(monto, token)
    PAG_SVC-->>PED_SVC: Pago aprobado ✓
    
    PED_SVC->>PED_DOM: new Pedido(ítems, total)
    PED_DOM-->>PED_SVC: Pedido validado
    
    PED_SVC->>PED_REPO: guardar(pedido)
    PED_REPO->>DB: INSERT INTO pedidos
    DB-->>PED_REPO: id generado
    PED_REPO-->>PED_SVC: {id, total, estado}
    
    PED_SVC-->>PED_CTRL: PedidoDTO
    PED_CTRL-->>WEB: 201 Created · JSON
    WEB-->>Cliente: ✅ Confirmación
```

---

## DIAGRAMA 4: Vista de Despliegue (Escalamiento Horizontal)

Muestra cómo se distribuye el monolito en producción con réplicas, load balancer y bases de datos.

```mermaid
flowchart LR
    NAV["🌐 Navegadores<br/>Cliente · Seller · Admin"]
    
    LB["⚖️ Load Balancer<br/>Nginx · HTTPS<br/>Distribuye tráfico"]

    subgraph INSTANCIAS["Instancias del Monolito (stateless)"]
        I1["Instance 1<br/>Node.js + Express"]
        I2["Instance 2<br/>Node.js + Express"]
        IN["Instance N<br/>Node.js + Express"]
    end

    PGP[("🗄️ PostgreSQL<br/>PRIMARY<br/>Escritura<br/>Replicación")]
    
    PGR[("🗄️ PostgreSQL<br/>READ REPLICA<br/>Lectura")]
    
    CACHE[("⚡ Redis<br/>Caché<br/>Consultas frecuentes")]
    
    EXT["🔗 Sistemas Externos<br/>Pago · Envío<br/>Facturación · ERP"]

    NAV -->|"HTTPS"| LB
    
    LB -->|"Round Robin"| I1
    LB -->|"Round Robin"| I2
    LB -->|"Round Robin"| IN

    I1 -->|"write"| PGP
    I2 -->|"write"| PGP
    IN -->|"write"| PGP

    I1 -.->|"read"| PGR
    I2 -.->|"read"| PGR
    IN -.->|"read"| PGR

    I1 --> CACHE
    I2 --> CACHE
    IN --> CACHE

    PGP -.->|"replicación<br/>continua"| PGR

    I1 --> EXT
    I2 --> EXT
    IN --> EXT

    classDef lb fill:#bbdefb,stroke:#1565c0,stroke-width:2px,color:#000
    classDef inst fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px,color:#000
    classDef db fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#000
    classDef cache fill:#e1bee7,stroke:#6a1b9a,stroke-width:2px,color:#000
    classDef ext fill:#eeeeee,stroke:#616161,stroke-width:2px,color:#000

    class LB lb
    class I1,I2,IN inst
    class PGP,PGR db
    class CACHE cache
    class EXT ext

    style INSTANCIAS fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
```
