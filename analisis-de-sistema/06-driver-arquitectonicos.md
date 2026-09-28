# Drivers Arquitectónicos — Marketplace de productos para mascotas


## Tabla de Drivers Arquitectónicos

| Driver | Problema que plantea | Decisión que responde |
| :--- | :--- | :--- |
| **DA01- Escalabilidad** | Aumentarán usuarios en campañas | Monolito modular con posibilidad de escalamiento horizontal |
| **DA02- Rendimiento** | Habrá alta concurrencia | Incorporar caché y optimizar comunicación/procesamiento |
| **DA03 -Seguridad** | Hay datos sensibles | Autenticación y autorización |
| **DA04 - Pago externo** | Hay que comunicarse con una pasarela | Integración mediante API y adaptadores |
| **DA05 -API REST** | Frontend/backend deben comunicarse mediante REST | Separar interfaz y backend mediante API REST |
| **DA06 - Mantenibilidad** | Cambios no deben afectar otros módulos | Modularidad + Clean Architecture |
---


## Matriz de relaciones
 
```mermaid
flowchart LR
    HU["Historias de Usuario"]
    RF["Requisitos Funcionales"]
    AC["Atributos de Calidad"]
    RC["Restricciones"]
    
    DA["DRIVERS ARQUITECTÓNICOS"]
    
    ARQ["Arquitectura del Sistema"]
 
    HU --> RF
    RF --> DA
    AC --> DA
    RC --> DA
    
    DA --> ARQ
 
    style HU fill:#bbdefb,color:#000
    style RF fill:#c8e6c9,color:#000
    style AC fill:#ffe0b2,color:#000
    style RC fill:#ffccbc,color:#000
    style DA fill:#f8bbd0,stroke:#c2185b,stroke-width:3px,color:#000
    style ARQ fill:#e1bee7,stroke:#7b1fa2,stroke-width:3px,color:#000
```

---

- **Escalabilidad:** Considerar load balancers, réplicas, caché
- **Rendimiento:** Optimizar queries, implementar índices, usar caché
- **Seguridad:** OAuth, JWT, HTTPS, encriptación, RBAC
- **Disponibilidad:** Redundancia, réplicas, health checks
- **Mantenibilidad:** Modularidad, documentación, pruebas, CI/CD


