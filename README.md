# TicketAR — Plataforma de Venta de Entradas · Mundial 2026

[![Backend CI](https://github.com/PzocikErwin/TicketAR-Mundial-UCP-2026/actions/workflows/backend-ci.yml/badge.svg)](https://github.com/PzocikErwin/TicketAR-Mundial-UCP-2026/actions/workflows/backend-ci.yml)
[![Frontend CI](https://github.com/PzocikErwin/TicketAR-Mundial-UCP-2026/actions/workflows/frontend-ci.yml/badge.svg)](https://github.com/PzocikErwin/TicketAR-Mundial-UCP-2026/actions/workflows/frontend-ci.yml)
[![Codacy Badge](https://app.codacy.com/project/badge/Grade/4167fcb406a44fb89c4cdc047a3eb7a8)](https://app.codacy.com)
![Node.js](https://img.shields.io/badge/Node.js-20%2B-339933?logo=nodedotjs&logoColor=white)
![NestJS](https://img.shields.io/badge/NestJS-11-E0234E?logo=nestjs&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-5-3178C6?logo=typescript&logoColor=white)
![Supabase](https://img.shields.io/badge/Supabase-PostgreSQL-3FCF8E?logo=supabase&logoColor=white)
![Playwright](https://img.shields.io/badge/Playwright-E2E-2EAD33?logo=playwright&logoColor=white)
![Jest](https://img.shields.io/badge/Jest-Unit_Tests-C21325?logo=jest&logoColor=white)

<p align="center">
  <a href="https://nestjs.com" target="_blank"><img src="https://cdn.simpleicons.org/nestjs/E0234E" alt="NestJS" width="40" /></a>
  <a href="https://nextjs.org" target="_blank"><img src="https://cdn.simpleicons.org/nextdotjs/000000" alt="Next.js" width="40" /></a>
  <a href="https://www.typescriptlang.org" target="_blank"><img src="https://cdn.simpleicons.org/typescript/3178C6" alt="TypeScript" width="40" /></a>
  <a href="https://supabase.com" target="_blank"><img src="https://cdn.simpleicons.org/supabase/3FCF8E" alt="Supabase" width="40" /></a>
  <a href="https://clerk.com" target="_blank"><img src="https://cdn.simpleicons.org/clerk/6C47FF" alt="Clerk" width="40" /></a>
  <a href="https://www.mercadopago.com" target="_blank"><img src="https://cdn.simpleicons.org/mercadopago/00B1EA" alt="Mercado Pago" width="40" /></a>
  <a href="https://playwright.dev" target="_blank"><img src="https://cdn.simpleicons.org/playwright/2EAD33" alt="Playwright" width="40" /></a>
  <a href="https://jestjs.io" target="_blank"><img src="https://cdn.simpleicons.org/jest/C21325" alt="Jest" width="40" /></a>
  <a href="https://railway.app" target="_blank"><img src="https://cdn.simpleicons.org/railway/0B0D0E" alt="Railway" width="40" /></a>
</p>

> Plataforma web para la compra segura de entradas del Mundial 2026. Diseñada para eliminar la reventa ilegal: cada entrada está vinculada al pasaporte del titular y se valida en la puerta.

---

## ¿Por qué está construido así?

El problema real que resuelve este sistema no es solo vender entradas — es garantizar que **quien compra es quien entra**, y que ningún asiento se venda dos veces. Esas dos restricciones guían cada decisión de arquitectura.

### Base de datos: Supabase (PostgreSQL)

Supabase provee PostgreSQL con RLS (Row Level Security), lo que permite definir reglas de acceso a nivel de tabla directamente en la base de datos, no solo en la capa de aplicación. La alternativa era gestionar una instancia de Postgres propia, pero en un proyecto de este alcance (6 sectores, ~64 partidos, concurrencia moderada) el overhead operacional no se justifica. Supabase también incluye las funciones de DB que usamos para recalcular el estado de los partidos sin round-trips adicionales.

### Autenticación: Clerk

Clerk se eligió sobre implementar JWT propio porque la validación de identidad es el núcleo del anti-fraude. Clerk provee un `userId` verificado por sesión que vinculamos a los datos de pasaporte en nuestra DB. Si el token es inválido, el guard de NestJS rechaza la request antes de llegar al servicio. Implementar esto desde cero con bcrypt + refresh tokens habría sido correcto, pero el costo de mantenimiento no agrega valor diferencial al proyecto.

### Ciclo de vida del ticket: State Pattern

Un ticket pasa por `RESERVADO → PAGADO → CANCELADO` y las transiciones tienen reglas estrictas: no se puede pagar un ticket expirado, no se puede cancelar uno ya pagado, no se puede generar un QR de uno reservado. En lugar de un `switch` por estado en cada método del servicio, cada estado es una clase con sus propias restricciones. El servicio no pregunta "¿en qué estado está?" — le delega la decisión al estado mismo.

```
ReservadoState.pagar()   → valida expiración → delega a PaymentsService
PagadoState.pagar()      → lanza BadRequestException inmediatamente
CanceladoState.cancelar() → lanza BadRequestException inmediatamente
```

### Pagos: Strategy Pattern

El sistema soporta dos flujos de pago: **Mercado Pago** (asíncrono — el usuario va a un checkout externo y vuelve vía webhook) y un **modo simulado** (síncrono — útil para tests y demos). Ambos implementan `PaymentStrategy`. El servicio de tickets no sabe cuál está activo — solo llama `processTicketPayment()`. Cambiar de proveedor de pago no requiere tocar el servicio de negocio.

### Comunicación entre módulos: Event Emitter (Observer)

Cuando un ticket se paga, el módulo de notificaciones tiene que enviar el QR por email. La dependencia directa `TicketsService → NotificationsService` crearía un acoplamiento circular (Notifications necesita datos de Tickets). En cambio, `EntradasService` emite un evento `ticket.pagado` y `NotificationsService` lo escucha. Ningún módulo conoce al otro.

### Reserva temporal: Cron Job

Los asientos se bloquean por 15 minutos al reservar. Si el usuario no paga, el stock tiene que volver a estar disponible. Implementamos esto con un `@Cron(EVERY_MINUTE)` que escanea reservas expiradas y hace rollback del stock + actualiza el estado del partido. La alternativa (TTL en Redis) implicaba agregar otro servicio al stack; el cron sobre la misma DB es suficiente para la escala actual y más simple de operar.

---

## Flujo principal

```
Usuario → reserva asientos (stock se decrementa en DB)
        → checkout (15 min para pagar)
           ├─ Pago simulado: ticket → PAGADO inmediatamente
           └─ Mercado Pago: usuario va al checkout externo
                           → webhook POST /payments/webhook
                           → backend verifica con MP API
                           → ticket → PAGADO
                           → evento `ticket.pagado`
                              └─ NotificationsService envía QR por email
```

---

## Estructura del proyecto

```
.
├── backend-nest/          # API REST — NestJS modular
│   └── src/
│       ├── tickets/       # Núcleo: reserva, estado, QR, expiración
│       ├── payments/      # Strategy Pattern: MP + simulado, webhook
│       ├── usuarios/      # Gestión de cuenta y validación de pasaporte
│       ├── matches/       # Partidos y disponibilidad
│       ├── stadium-sectors/ # Sectores y precios
│       ├── notifications/ # Envío de QR por email (event-driven)
│       ├── stats/         # Panel administrativo
│       └── common/        # Guards, decorators, enums, Supabase client
├── frontend-client/       # Next.js (App Router) + Tailwind + Clerk
└── docs/                  # Documentación técnica y diagramas
```

---

## Patrones aplicados

| Patrón | Dónde | Por qué |
|---|---|---|
| **State** | `tickets/states/` | Transiciones de ticket sin condicionales dispersos |
| **Strategy** | `payments/strategies/` | Intercambiar proveedor de pago sin tocar negocio |
| **Repository** | `tickets/repositories/` | Aislar la capa de datos para poder testear sin DB |
| **Observer / Event-Driven** | `EventEmitter2` en AppModule | Desacoplar notificaciones de la lógica de pago |
| **Factory** | `TicketStateFactory` | Instanciar el estado correcto dado un string de DB |

---

## Diagrama de clases

<p align="center">
  <a href="docs/diagrams/ticketar-class-diagram.svg">
    <img src="docs/diagrams/ticketar-class-diagram.svg" alt="Diagrama UML de clases de TicketAR" width="100%" />
  </a>
</p>

Fuente editable: [`docs/diagrams/ticketar-class-diagram.mmd`](docs/diagrams/ticketar-class-diagram.mmd)

---

## ¿Qué se rompería con 100× más tráfico?

**1. El cron de expiración de reservas** es el cuello de botella más inmediato. Actualmente corre cada minuto en el proceso principal y hace un query + N updates (uno por ticket expirado). Con decenas de miles de reservas simultáneas, ese loop compite con requests normales por el connection pool de Supabase. La solución es mover la lógica a una función de base de datos (Postgres scheduled function o trigger con `pg_cron`) que opere directamente sobre los datos sin pasar por la aplicación.

**2. El webhook de Mercado Pago no tiene lock distribuido.** Si MP envía la misma notificación dos veces en paralelo (lo cual hace bajo carga), dos workers podrían intentar marcar el mismo ticket como pagado simultáneamente. La solución es un advisory lock de Postgres (`SELECT pg_advisory_xact_lock(ticket_id)`) o un upsert condicional que ignore si el estado ya es `PAGADO`.

**3. Stock sin transacción atómica.** El decremento de stock actual es un `UPDATE ... SET stock = stock - N WHERE stock >= N`. Bajo alta concurrencia, dos requests para el mismo sector podrían leer el mismo stock disponible y ambas pasar la validación. La solución es mover el decremento a una función de DB que ejecute el check y el update en una sola transacción serializable.

---

## Seguridad

- `ValidationPipe` global con `whitelist: true` — rechaza cualquier campo no declarado en el DTO
- Guards de autenticación (Clerk) y autorización por rol (`ADMINISTRADOR`) en rutas sensibles
- El QR se genera solo para tickets en estado `PAGADO` — no hay forma de obtener un QR antes de confirmar el pago
- Máximo 6 entradas por cuenta por partido, validado en el servicio (no solo en el frontend)
- `.env` nunca commiteado — ver `.env.example` para las variables requeridas

---

## Ejecutar localmente

### Requisitos

- Node.js 20+
- pnpm 9+

```bash
npm install -g pnpm
```

### Variables de entorno

Copiar `.env.example` en cada app y completar con tus credenciales de Supabase, Clerk y Mercado Pago:

```bash
cp .env.example backend-nest/.env
cp frontend-client/.env.example frontend-client/.env.local
```

### Backend

```bash
cd backend-nest
pnpm install
pnpm run start:dev   # http://localhost:3000
```

### Frontend

```bash
cd frontend-client
pnpm install
pnpm run dev         # http://localhost:3001
```

---

## Tests y CI

```bash
# Backend — unitarios
cd backend-nest && pnpm run test

# Backend — E2E (Playwright)
cd backend-nest && pnpm run test:playwright

# Frontend — E2E y componentes (Playwright)
cd frontend-client && pnpm run test
```

El pipeline de GitHub Actions corre en cada push a `main`: build, lint y tests unitarios para backend y frontend.

---

## Convención de commits

- `feat:` nueva funcionalidad
- `fix:` corrección de errores
- `docs:` cambios de documentación
- `refactor:` mejora interna sin cambiar comportamiento
- `chore:` tareas de mantenimiento

---

## Seguridad y vulnerabilidades

Para reporte responsable de vulnerabilidades, ver [SECURITY.md](SECURITY.md).

---

<p align="center">
  <img src="https://contrib.rocks/image?repo=JuanmaInv/TicketAR---Mundial---UCP---2026-" />
</p>
