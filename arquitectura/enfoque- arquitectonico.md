# Enfoque Arquitectónico — Marketplace de productos para mascotas

## Objetivo

Definir **cómo se organizan las responsabilidades y las dependencias internas** de cada módulo, respondiendo a la pregunta: **¿cómo estructuramos el código dentro del monolito?** La respuesta es **Clean Architecture**, que organiza cada módulo en capas concéntricas donde las dependencias siempre apuntan hacia el núcleo (dominio).

---

## Definición: Clean Architecture

**Clean Architecture** es un enfoque que organiza el código en capas independientes, donde:

- El **núcleo (dominio)** contiene la lógica de negocio pura, sin dependencias externas.
- Las capas exteriores dependen de las capas interiores, **nunca al revés**.
- Los casos de uso (aplicación) orquestan el dominio y comunican con la infraestructura.
- La presentación (controladores y rutas) es la capa más externa.

### Ventajas para este proyecto

1. **Independencia de frameworks:** la lógica de negocio no depende de Express, ORM o bases de datos.
2. **Testabilidad:** los casos de uso se prueban sin mocks complejos de dependencias externas.
3. **Mantenibilidad (DA06):** cambios en la base de datos o en el framework no afectan la lógica central.
4. **Escalabilidad:** es fácil agregar nuevas formas de acceder al negocio (API REST, webhooks, CLI) sin duplicar lógica.

---

## Capas de Clean Architecture

```
┌─────────────────────────────────────────────────┐
│         PRESENTACIÓN (routes, controllers)       │  Recibe HTTP, valida entrada
│         Dependencias: → Aplicación               │
├─────────────────────────────────────────────────┤
│    APLICACIÓN (cases, orchestration)             │  Orquesta dominio + infra
│    Dependencias: → Dominio, → Infraestructura    │
├─────────────────────────────────────────────────┤
│      DOMINIO (entities, value objects)           │  Reglas del negocio puras
│      Dependencias: → nada (independiente)        │
├─────────────────────────────────────────────────┤
│  INFRAESTRUCTURA (ORM, APIs, repositorios)       │  Detalles técnicos
│  Dependencias: → Dominio, → Aplicación           │
└─────────────────────────────────────────────────┘
```

---

## Capas detalladas

### 1. Dominio (Domain Layer)

**Responsabilidad:** Contener las reglas de negocio puras, sin acoplamiento a tecnología externa.

**Contenido:**
- **Entidades (Entities):** Objetos de negocio que representan conceptos centrales (Usuario, Producto, Pedido).
- **Objetos de Valor (Value Objects):** Objetos inmutables que representan valores sin identidad propia (Precio, Dirección, Email).
- **Casos de Uso (Use Cases):** Lógica de negocio de nivel alto (RegistrarUsuario, CrearPedido, AplicarDescuento).

**Características:**
- No importa librerías externas (Express, Sequelize, etc.).
- No conoce detalles de cómo se persisten los datos.
- Reglas de validación y decisiones del negocio son claras y testables.

**Ejemplo — Módulo Usuarios:**

```javascript
// src/modules/usuarios/domain/Usuario.js
class Usuario {
  constructor(id, email, contraseña, rol) {
    if (!this.esEmailValido(email)) throw new Error('Email inválido');
    this.id = id;
    this.email = email;
    this.contraseña = contraseña;  // en la realidad, hasheada
    this.rol = rol;  // 'cliente' | 'seller' | 'admin'
  }

  esEmailValido(email) {
    return /^[\w.-]+@[\w.-]+\.\w+$/.test(email);
  }

  puedeCancelarPedido() {
    return this.rol === 'cliente' || this.rol === 'admin';
  }
}
```

```javascript
// src/modules/usuarios/domain/RegistrarUsuario.js (Use Case)
class RegistrarUsuario {
  constructor(usuarioRepository) {
    this.usuarioRepository = usuarioRepository;
  }

  async ejecutar(email, contraseña, rol) {
    // Validar que el email no esté ya registrado
    const usuarioExistente = await this.usuarioRepository.obtenerPorEmail(email);
    if (usuarioExistente) {
      throw new Error('El email ya está registrado');
    }

    // Crear la entidad de dominio
    const nuevoUsuario = new Usuario(
      null,  // id generado por la BD
      email,
      contraseña,
      rol
    );

    // Persistir mediante el repositorio
    const usuarioGuardado = await this.usuarioRepository.guardar(nuevoUsuario);
    return usuarioGuardado;
  }
}
```

### 2. Aplicación (Application Layer)

**Responsabilidad:** Orquestar el dominio y coordinar con la infraestructura. Contiene los casos de uso en forma de servicios de aplicación.

**Contenido:**
- **Servicios de Aplicación:** Orquestadores que usan casos de uso del dominio y repositorios.
- **DTOs (Data Transfer Objects):** Objetos para transportar datos entre capas sin exponer entidades de dominio.
- **Excepciones de aplicación:** Errores específicos del negocio (UsuarioNoEncontrado, StockInsuficiente).

**Características:**
- Delega la lógica pura al dominio.
- Coordina múltiples casos de uso si es necesario.
- No conoce detalles de Express, ORM o bases de datos específicas.

**Ejemplo — Módulo Usuarios:**

```javascript
// src/modules/usuarios/application/RegistrarUsuarioService.js
class RegistrarUsuarioService {
  constructor(usuarioRepository, emailNotifier) {
    this.usuarioRepository = usuarioRepository;
    this.emailNotifier = emailNotifier;
    this.registrarUsuarioUseCase = new RegistrarUsuario(usuarioRepository);
  }

  async registrar(email, contraseña, rol) {
    try {
      const nuevoUsuario = await this.registrarUsuarioUseCase.ejecutar(
        email,
        contraseña,
        rol
      );

      // Notificar (infraestructura)
      await this.emailNotifier.enviarBienvenida(email);

      return {
        id: nuevoUsuario.id,
        email: nuevoUsuario.email,
        rol: nuevoUsuario.rol,
      };
    } catch (error) {
      if (error.message === 'El email ya está registrado') {
        throw new UsuarioYaExisteError(email);
      }
      throw error;
    }
  }
}
```

### 3. Presentación (Presentation Layer)

**Responsabilidad:** Recibir peticiones HTTP, validar entrada y responder en JSON. Traducir entre HTTP y los DTOs de aplicación.

**Contenido:**
- **Routes:** Definir las rutas y vincularlas a controladores.
- **Controllers:** Manejar la petición HTTP, invocar servicios de aplicación, responder.

**Características:**
- No contiene lógica de negocio (eso está en Dominio y Aplicación).
- Solo traduce HTTP ↔ lógica de negocio.
- Maneja errores y serializa respuestas.

**Ejemplo — Módulo Usuarios:**

```javascript
// src/modules/usuarios/presentation/usuarios.controller.js
class UsuariosController {
  constructor(registrarUsuarioService) {
    this.registrarUsuarioService = registrarUsuarioService;
  }

  async registrar(req, res, next) {
    try {
      const { email, contraseña, rol } = req.body;

      // Invocar el servicio de aplicación
      const usuarioDTO = await this.registrarUsuarioService.registrar(
        email,
        contraseña,
        rol
      );

      res.status(201).json({
        success: true,
        data: usuarioDTO,
      });
    } catch (error) {
      next(error);  // El middleware de errores lo maneja
    }
  }
}
```

```javascript
// src/modules/usuarios/presentation/usuarios.routes.js
const express = require('express');
const router = express.Router();

module.exports = (controller) => {
  router.post('/registro', (req, res, next) => controller.registrar(req, res, next));
  // más rutas...
  return router;
};
```

### 4. Infraestructura (Infrastructure Layer)

**Responsabilidad:** Implementar detalles técnicos: acceso a bases de datos, llamadas a APIs externas, caché, etc.

**Contenido:**
- **Repositorios:** Implementación concreta del contrato de persistencia (ORM, SQL directo).
- **Adaptadores:** Integración con sistemas externos (pasarela de pagos, servicio de envíos).
- **Clientes HTTP:** Para consumir APIs externas.
- **Notificadores:** Envío de emails, SMS, notificaciones.

**Características:**
- Conoce Express, Sequelize, Axios, Redis, etc.
- Implementa interfaces definidas por la aplicación.
- Puede reemplazarse sin afectar el negocio.

**Ejemplo — Módulo Usuarios:**

```javascript
// src/modules/usuarios/infrastructure/UsuarioRepository.js (ORM Sequelize)
const { Usuario: UsuarioModel } = require('../../../shared/db/models');

class UsuarioRepository {
  async guardar(usuarioDominio) {
    const usuarioModel = await UsuarioModel.create({
      email: usuarioDominio.email,
      contraseña: usuarioDominio.contraseña,  // hasheada
      rol: usuarioDominio.rol,
    });

    // Retornar la entidad de dominio (no el model de Sequelize)
    return new Usuario(
      usuarioModel.id,
      usuarioModel.email,
      usuarioModel.contraseña,
      usuarioModel.rol
    );
  }

  async obtenerPorEmail(email) {
    const usuarioModel = await UsuarioModel.findOne({ where: { email } });
    if (!usuarioModel) return null;

    return new Usuario(
      usuarioModel.id,
      usuarioModel.email,
      usuarioModel.contraseña,
      usuarioModel.rol
    );
  }
}
```

```javascript
// src/modules/usuarios/infrastructure/EmailNotifier.js
const nodemailer = require('nodemailer');

class EmailNotifier {
  constructor(transportConfig) {
    this.transporter = nodemailer.createTransport(transportConfig);
  }

  async enviarBienvenida(email) {
    await this.transporter.sendMail({
      to: email,
      subject: 'Bienvenido al Marketplace',
      html: '<h1>¡Gracias por registrarte!</h1>',
    });
  }
}
```

---

## Aplicación al proyecto Marketplace

### Estructura de carpetas por módulo

Cada módulo en `src/modules/<módulo>/` sigue este patrón:

```text
src/modules/usuarios/
├── domain/
│   ├── Usuario.js                  # Entidad
│   ├── RegistrarUsuario.js         # Use Case
│   └── ...
├── application/
│   ├── RegistrarUsuarioService.js  # Servicio de aplicación
│   ├── dtos/
│   │   └── UsuarioDTO.js
│   └── exceptions/
│       └── UsuarioYaExisteError.js
├── presentation/
│   ├── usuarios.controller.js      # Controlador
│   └── usuarios.routes.js          # Rutas
└── infrastructure/
    ├── UsuarioRepository.js        # Implementación con ORM
    ├── EmailNotifier.js            # Notificaciones
    └── ...
```

### Ejemplo completo: Módulo Pedidos (RF05 — Generar pedido)

El caso de uso "generar pedido" requiere interacción entre módulos. Con Clean Architecture:

**Dominio (Pedidos):**
```javascript
// src/modules/pedidos/domain/Pedido.js
class Pedido {
  constructor(id, clienteId, ítems, total, estado) {
    this.validarÍtems(ítems);
    this.id = id;
    this.clienteId = clienteId;
    this.ítems = ítems;
    this.total = total;
    this.estado = 'pendiente';
  }

  validarÍtems(ítems) {
    if (!ítems || ítems.length === 0) {
      throw new Error('El pedido debe tener al menos un ítem');
    }
  }

  puedeSerCancelado() {
    return this.estado === 'pendiente' || this.estado === 'procesando';
  }
}
```

**Aplicación (Pedidos):**
```javascript
// src/modules/pedidos/application/CrearPedidoService.js
class CrearPedidoService {
  constructor(
    pedidoRepository,
    carritoService,       // servicio de otro módulo
    catalogoService,      // servicio de otro módulo
    pagosService,         // servicio de otro módulo
    envioAdapter,
    facturacionAdapter
  ) {
    this.pedidoRepository = pedidoRepository;
    this.carritoService = carritoService;
    this.catalogoService = catalogoService;
    this.pagosService = pagosService;
    this.envioAdapter = envioAdapter;
    this.facturacionAdapter = facturacionAdapter;
  }

  async crearPedido(clienteId, monto, tokenPago) {
    // 1. Obtener carrito (módulo carrito)
    const carrito = await this.carritoService.obtenerCarrito(clienteId);
    if (!carrito || carrito.ítems.length === 0) {
      throw new CarritoVacíoError();
    }

    // 2. Reservar stock (módulo catálogo)
    await this.catalogoService.reservarStock(carrito.ítems);

    // 3. Procesar pago (módulo pagos)
    const pago = await this.pagosService.procesar(monto, tokenPago);
    if (!pago.exitoso) {
      throw new PagoFallidoError();
    }

    // 4. Crear la entidad de dominio
    const nuevoPedido = new Pedido(
      null,
      clienteId,
      carrito.ítems,
      monto,
      'pendiente'
    );

    // 5. Persistir
    const pedidoGuardado = await this.pedidoRepository.guardar(nuevoPedido);

    // 6. Coordinar integraciones (infraestructura)
    await this.envioAdapter.registrar(pedidoGuardado);
    await this.facturacionAdapter.emitir(pedidoGuardado);

    return {
      id: pedidoGuardado.id,
      clienteId: pedidoGuardado.clienteId,
      total: pedidoGuardado.total,
      estado: pedidoGuardado.estado,
    };
  }
}
```

**Presentación (Pedidos):**
```javascript
// src/modules/pedidos/presentation/pedidos.controller.js
class PedidosController {
  constructor(crearPedidoService) {
    this.crearPedidoService = crearPedidoService;
  }

  async crear(req, res, next) {
    try {
      const { monto, tokenPago } = req.body;
      const clienteId = req.usuario.id;

      const pedidoDTO = await this.crearPedidoService.crearPedido(
        clienteId,
        monto,
        tokenPago
      );

      res.status(201).json({
        success: true,
        data: pedidoDTO,
      });
    } catch (error) {
      next(error);
    }
  }
}
```

---

## Reglas de Dependencia en Clean Architecture

| Regla | Explicación | Válido | Inválido |
|---|---|---|---|
| **Las capas exteriores pueden depender de las interiores.** | El Dominio no conoce a Infraestructura, pero Infraestructura implementa interfaces del Dominio. | `infraestructura → dominio` | `dominio → infraestructura` |
| **Las capas interiores son independientes.** | El Dominio no importa nada externo; es 100% puro. | `dominio → nada` | `dominio → aplicación` |
| **La Presentación delega al Dominio y Aplicación, no a Infraestructura.** | El controlador llama al servicio de aplicación; el servicio coordina todo. | `controller → service → domain` | `controller → repository` |
| **Los objetos que cruzan capas son DTOs, no entidades de dominio.** | Evita exponer la estructura interna del negocio. | `controller → UsuarioDTO` | `controller → Usuario (entidad)` |

---

## Inyección de Dependencias

Clean Architecture requiere que las dependencias se inyecten, no se instancien dentro de cada capa. Esto asegura que el cambio de implementaciones (por ejemplo, Base de Datos) no requiera cambiar el código de negocio.

```javascript
// src/modules/usuarios/index.js (Configuración del módulo)
const RegistrarUsuarioService = require('./application/RegistrarUsuarioService');
const UsuariosController = require('./presentation/usuarios.controller');
const UsuarioRepository = require('./infrastructure/UsuarioRepository');
const EmailNotifier = require('./infrastructure/EmailNotifier');

// Las dependencias se instancian una sola vez (en startup)
const usuarioRepository = new UsuarioRepository();
const emailNotifier = new EmailNotifier(emailConfig);
const registrarUsuarioService = new RegistrarUsuarioService(
  usuarioRepository,
  emailNotifier
);
const usuariosController = new UsuariosController(registrarUsuarioService);

module.exports = {
  controller: usuariosController,
  repository: usuarioRepository,
};
```

---

## Comparación: Antes (Guía 02) vs. Ahora (Clean Architecture)

| Aspecto | Arquitectura 3 capas (Guía 02) | Clean Architecture |
|---|---|---|
| Organización | Routes → Controllers → Services → Repositories (capas horizontales) | Cada módulo tiene sus propias capas (vertical slices) |
| Dónde va la lógica | Servicio + Repository (mezclado) | Dominio (puro) + Aplicación (orquestación) |
| Dependencias | Services importan Repositories directamente | Infraestructura implementa interfaces de Dominio y Aplicación |
| Testabilidad | Difícil; necesita mocks del ORM | Fácil; se prueba el Dominio sin dependencias |
| Escalabilidad | Cambiar ORM requiere tocar Services | Cambiar ORM solo toca Infraestructura |

---

## Relación con los Drivers Arquitectónicos

| Driver | ¿Cómo Clean Architecture lo atiende? |
|---|---|
| **DA01 — Escalabilidad** | Código independiente de framework permite reescalar la lógica en diferentes plataformas. |
| **DA02 — Rendimiento** | Dominio puro es rápido; la caché se implementa en Infraestructura sin tocar la lógica. |
| **DA03 — Seguridad** | Validaciones de negocio están en Dominio; no se pueden saltear en Presentación. |
| **DA06 — Mantenibilidad** | Cambios en Infraestructura no afectan Dominio; cada capa tiene responsabilidad única. |

---

## Relación con ADR-002

**ADR-002 — Clean Architecture**

Clean Architecture es el patrón elegido para organizar internamente cada módulo del monolito. Cumple con:

- ✅ Separación de responsabilidades entre capas.
- ✅ Dependencias apuntando al dominio.
- ✅ Independencia de frameworks y bases de datos.
- ✅ Testabilidad sin mocks complejos.

---

## Próximos pasos

**Paso 6:** Crear los **Patrones de Diseño** específicos si son necesarios (Strategy para diferentes tipos de envío, Factory para crear objetos de dominio, Decorator para agregar funcionalidad sin mutar entidades).

**Paso 7:** Documentar la estructura de **Testing** (pruebas unitarias en Dominio, integración en Aplicación, E2E en Presentación).

**Paso 8:** Definir **Convenciones de Nombrado** específicas del proyecto (cómo se llaman los dtos, excepciones, interfaces).