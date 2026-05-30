# Nightscout API v3 — Reimplementación

## Contexto

Proyecto personal de un paciente diabético que usa Nightscout + AndroidAPS diariamente.
Objetivo: reescribir la API v3 de Nightscout con un stack moderno manteniendo compatibilidad
total con los clientes existentes (AndroidAPS NSClient v3, xDrip+).

No es un fork de Nightscout ni pretende sustituirlo para la comunidad. Es un proyecto
de aprendizaje del ecosistema Node.js moderno aplicado a un problema real y personal.

---

## Decisiones de Arquitectura

### ADR-001 — Runtime: Bun

**Decisión:** Bun como runtime en lugar de Node.js.

**Por qué:**
- Arranque más rápido (~4x vs Node) — relevante para Fly.io donde los contenedores se duermen
- Menor consumo de memoria en idle — importante para un servidor personal
- Compatible con el ecosistema Node.js — no hay lock-in
- Package manager 25x más rápido que npm

**Alternativas descartadas:**
- Node.js: válido pero sin ventajas sobre Bun para este caso
- Deno: nicho, nunca despegó del todo para servidores persistentes

---

### ADR-002 — Framework: Fastify

**Decisión:** Fastify como framework HTTP.

**Por qué:**
- Estándar de industria para APIs REST en Node.js (~10M descargas semanales, desde 2016)
- Battle-tested para servidores persistentes — exactamente el caso de uso
- Ecosistema maduro: auth, swagger, websockets, rate limiting, todo tiene plugin oficial
- TypeScript support sólido
- `@fastify/swagger` + `zod-to-json-schema` permite replicar el contrato API v3 con validación automática

**Alternativas descartadas:**
- Elysia: performance superior pero Bun-native, lock-in de runtime. Performance no es prioridad.
- Hono: mejor DX TypeScript, más interoperable, pero diseñado para edge/serverless first.
  Para un servidor persistente Fastify es más apropiado.
- NestJS: demasiado opinionado, boilerplate excesivo, overkill para este scope.
- Express: legacy, nadie lo elige para proyectos nuevos.

**Nota:** La decisión fue Hono → Fastify tras evaluar que el argumento de interoperabilidad
de runtime no aplica para un servidor persistente en Fly.io.

---

### ADR-003 — Base de datos: TimescaleDB (PostgreSQL)

**Decisión:** TimescaleDB en lugar de MongoDB.

**Por qué:**
- Los datos de glucosa son series temporales puras — lecturas cada 5 minutos
- Queries temporales (promedios por hora, rango de 24h, time-in-range) son triviales con `time_bucket()`
- Las mismas queries en MongoDB requieren aggregation pipelines complejos y lentos
- Compresión automática de datos históricos (~90% ahorro de espacio)
- Agregados continuos precalculados sin coste en query time
- Es PostgreSQL — SQL estándar, sin aprender nueva tecnología
- Retención automática configurable

**Alternativas descartadas:**
- MongoDB: el proyecto original lo usa pero no está optimizado para series temporales
- PostgreSQL sin TimescaleDB: válido pero sin las ventajas de hypertables y compresión
- InfluxDB: diseñado para métricas de infraestructura, menos flexible para el modelo de datos

---

### ADR-004 — ORM: Drizzle

**Decisión:** Drizzle ORM.

**Por qué:**
- TypeScript-first — el schema se define en TypeScript, los tipos se infieren automáticamente
- Sin magia — el SQL generado es predecible y legible
- Cercano al SQL real — importante para aprovechar features de TimescaleDB
- Ligero y compatible con Bun desde el primer día
- Las migraciones son explícitas — tú controlas exactamente qué SQL se ejecuta (crítico para datos médicos)

**Alternativas descartadas:**
- Prisma: schema en lenguaje propio (.prisma), bundle grande, historial de problemas con Bun,
  abstrae demasiado el SQL (pierde features de TimescaleDB)
- TypeORM: decoradores, magic, difícil de predecir el SQL generado

---

### ADR-005 — Schema: ENUMs para valores fijos

**Decisión:** Usar PostgreSQL ENUMs para campos con valores conocidos y estables.

**Por qué:**
- Almacenamiento: 4 bytes vs 8-15 bytes por fila como TEXT
- Con ~105.000 lecturas de glucosa al año, el ahorro es significativo
- Validación a nivel de base de datos sin código extra
- Los valores llevan años sin cambiar (direction, units, glucose_type)

**ENUMs definidos:**
- `glucose_reading_type`: 'sgv' | 'mbg'
- `glucose_direction`: 'DoubleDown' | 'SingleDown' | 'FortyFiveDown' | 'Flat' | 'FortyFiveUp' | 'SingleUp' | 'DoubleUp' | 'NOT COMPUTABLE' | 'RATE OUT OF RANGE'
- `glucose_units`: 'mg/dl' | 'mmol/l'
- `treatment_event`: todos los eventTypes del API v3
- `glucose_source`: 'Finger' | 'Sensor' | 'Manual'
- `food_type`: 'food' | 'quickpick'

**Consideración:** `treatment_event` tiene más movimiento histórico. Si se añade un tipo nuevo
requiere `ALTER TYPE ADD VALUE` — no transaccional pero no bloquea la tabla.

---

### ADR-006 — Tabla única para lecturas de glucosa

**Decisión:** Una sola tabla `glucose_readings` para SGV (sensor) y MBG (manual).

**Por qué:**
- Conceptualmente son lo mismo: una lectura de glucosa en un punto en el tiempo
- La única diferencia es la fuente (sensor vs dedo)
- Una sola tabla simplifica todas las queries del dashboard
- Columnas SGV-específicas (direction, noise, filtered, unfiltered, rssi) quedan NULL para MBG

**Discriminador:** columna `type` con ENUM `glucose_reading_type`.

---

### ADR-007 — Autenticación: JWT con scopes

**Decisión:** JWT con scopes diferenciados por tipo de actor. Sin capa de compatibilidad legacy (ACL).

**Por qué:**
- AndroidAPS con NSClient v3 ya usa JWT via `/api/v2/authorization/request/:subject`
- xDrip+ moderno también soporta este mecanismo
- No tiene sentido mantener compatibilidad con `API-SECRET` header (SHA1) — es inseguro y los
  clientes que nos importan ya no lo necesitan
- JWT permite scopes granulares por tipo de actor

**Tipos de token y scopes:**
- `owner`: acceso total
- `device`: entries:write, devicestatus:write, treatments:write — para CGM bridges y bombas
- `caregiver`: entries:read, treatments:read — solo lectura
- `integration`: scoped por recurso — para IFTTT, HomeKit, etc.

**Token via query param `?token=`:** soportado como fallback para integraciones que no pueden
setear headers HTTP. Documentado explícitamente como menos seguro (URLs se loguean).
Nunca recomendado para el frontend.

**Refresh tokens:** opacos, guardados en Redis, revocables. Access tokens JWT de 15 minutos.
Tokens de dispositivo JWT de larga duración (1 año) con scope mínimo.

---

### ADR-008 — Contrato API: Nightscout API v3

**Decisión:** Replicar exactamente el contrato del API v3 de Nightscout.

**Por qué:**
- AndroidAPS NSClient v3 y xDrip+ funcionan sin ningún cambio de configuración
- Todos los frontends que reportan via API v3 (Nightscout Reporter, etc.) siguen funcionando
- El fichero `lib/api3/swagger.json` del repo original es la spec de referencia

**Endpoints MVP (suficientes para que AAPS funcione):**
- `POST /api/v3/entries` — AAPS sube lecturas de glucosa
- `GET  /api/v3/entries` — clientes leen glucosa
- `POST /api/v3/treatments` — AAPS sube boluses, carbos, basales
- `GET  /api/v3/treatments` — clientes leen tratamientos
- `POST /api/v3/devicestatus` — AAPS sube estado de bomba/loop
- `GET  /api/v3/devicestatus` — clientes leen estado
- `GET  /api/v2/authorization/request/:subject` — auth para NSClient v3
- `WebSocket` — updates en tiempo real

**Scope excluido del MVP:** frontend, sistema de plugins, notificaciones visuales, API v1/v2
de endpoints legacy.

---

### ADR-009 — Sistema de eventos: emit siempre, decidir en worker

**Decisión:** La API emite siempre un evento (`glucose.received`, `treatment.created`, etc.)
sin saber si hay que notificar. Los workers deciden.

**Por qué:**
- Separación de responsabilidades: la API no conoce las reglas de negocio de notificaciones
- Open/Closed: añadir nuevos comportamientos (HomeKit, agregados, ML) es un nuevo listener,
  cero cambios en la API
- El WebSocket y el sistema de notificaciones son dos listeners independientes del mismo evento

---

### ADR-010 — Cola de notificaciones: BullMQ sobre Redis

**Decisión:** BullMQ para procesar notificaciones de forma asíncrona.

**Por qué:**
- Las alarmas de hipoglucemia son críticas — no pueden perderse si Pushover está caído
- Reintentos automáticos con backoff exponencial
- Prioridades: hipoglucemia urgente salta la cola por delante de batería baja
- No bloquea la ingesta de datos: AAPS sigue escribiendo aunque el worker esté procesando
- Reutiliza el Redis ya necesario para auth — no es infraestructura adicional

**Alternativas descartadas:**
- Procesamiento síncrono: si el servicio de notificaciones falla, se pierde la alarma
- `setTimeout`/`setInterval`: no persiste entre reinicios del proceso
- RabbitMQ/Kafka: overkill, infraestructura extra innecesaria

---

### ADR-011 — Cache y estado: Redis

**Decisión:** Redis para tres usos concretos.

**Usos:**
1. Estado de refresh tokens (revocación de sesiones)
2. Rate limiting por token (dispositivos pueden hacer flood accidental)
3. Backend de BullMQ (colas de notificaciones)

---

### ADR-012 — Frontend: desacoplado, fuera del scope inicial

**Decisión:** El backend es API pura. No sirve ficheros estáticos ni tiene lógica de presentación.
El frontend no es parte del MVP.

**Por qué:**
- Ya existen frontends que consumen la API v3 (Nightscout Reporter, etc.)
- Permite desplegar frontend en CDN (Cloudflare Pages) independientemente del backend
- El objetivo inicial es que AndroidAPS funcione — no hay necesidad de frontend propio

---

### ADR-013 — Audit log

**Decisión:** Tabla append-only `audit_log` como hypertable en TimescaleDB.

**Por qué:**
- Datos médicos — trazabilidad de quién escribió qué y cuándo
- Append-only: nunca se modifica, solo se inserta
- Hypertable: misma eficiencia de TimescaleDB para queries temporales

---

## Stack Completo

```
Runtime      Bun
Framework    Fastify + @fastify/swagger + @fastify/websocket + @fastify/jwt
Validación   Zod + @fastify/type-provider-zod
Base datos   TimescaleDB (PostgreSQL)
ORM          Drizzle
Cache/Queue  Redis (Upstash o Fly.io Redis)
Cola         BullMQ
Despliegue   Fly.io (ya configurado en el fork)
```

---

## Plan de Implementación

### Fase 1 — Proyecto base

- [ ] Inicializar proyecto Bun + TypeScript (strict mode)
- [ ] Configurar Fastify con type-provider-zod
- [ ] Configurar ESLint + Prettier
- [ ] Dockerfile optimizado para Bun
- [ ] Variables de entorno tipadas con Zod

### Fase 2 — Schema de base de datos

- [ ] Configurar conexión TimescaleDB con Drizzle
- [ ] Definir ENUMs PostgreSQL
- [ ] Tabla `glucose_readings` (hypertable)
- [ ] Tabla `treatments` (hypertable)
- [ ] Tabla `device_status` (hypertable)
- [ ] Tabla `profiles`
- [ ] Tabla `food`
- [ ] Tabla `activity` (hypertable)
- [ ] Tabla `audit_log` (hypertable append-only)
- [ ] Sistema de migraciones con Drizzle
- [ ] Políticas de compresión y retención en TimescaleDB

### Fase 3 — Autenticación

- [ ] Endpoint `GET /api/v2/authorization/request/:subject`
- [ ] Middleware JWT con verificación de scopes
- [ ] Soporte `Authorization: Bearer` header
- [ ] Soporte `?token=` query param (documentado como inseguro)
- [ ] Refresh tokens en Redis
- [ ] Rate limiting por token con Redis

### Fase 4 — API v3 core (MVP para AAPS)

- [ ] `POST /api/v3/entries`
- [ ] `GET  /api/v3/entries` (con filtros, paginación, ordenación del swagger)
- [ ] `POST /api/v3/treatments`
- [ ] `GET  /api/v3/treatments`
- [ ] `POST /api/v3/devicestatus`
- [ ] `GET  /api/v3/devicestatus`
- [ ] Validación de schemas Zod contra swagger.json original
- [ ] Audit log en cada escritura

### Fase 5 — WebSockets

- [ ] Configurar @fastify/websocket
- [ ] Canal `glucose` — broadcast en cada nueva lectura
- [ ] Canal `treatments` — broadcast en cada nuevo tratamiento
- [ ] Autenticación en el handshake WebSocket

### Fase 6 — Sistema de eventos y notificaciones

- [ ] Event bus interno (EventEmitter tipado)
- [ ] Eventos: `glucose.received`, `treatment.created`, `devicestatus.updated`
- [ ] Configurar Redis + BullMQ
- [ ] Worker de notificaciones con prioridades
- [ ] Reglas de alarma: hipoglucemia urgente, hipoglucemia, hiperglucemia, pérdida de señal
- [ ] Integración Pushover
- [ ] Integración NTFY (alternativa open-source a Pushover)

### Fase 7 — Endpoints restantes API v3

- [ ] `GET/POST /api/v3/food`
- [ ] `GET/POST /api/v3/profile`
- [ ] `GET/POST /api/v3/activity`
- [ ] `GET /api/v3/settings`
- [ ] `GET /api/v1/status` (algunos clientes lo consultan)

### Fase 8 — Validación de contrato

- [ ] Tests de contrato contra swagger.json original
- [ ] Test de integración end-to-end con AndroidAPS real
- [ ] Test de integración con xDrip+

---

## Estructura de carpetas propuesta

```
src/
├── config/          # Variables de entorno, configuración
├── db/
│   ├── schema/      # Drizzle schema (enums, tables)
│   └── migrations/  # Migraciones SQL
├── modules/
│   ├── auth/        # JWT, tokens, scopes
│   ├── entries/     # glucose_readings endpoints
│   ├── treatments/  # treatments endpoints
│   ├── devicestatus/
│   ├── food/
│   ├── profile/
│   └── activity/
├── events/          # Event bus tipado
├── workers/
│   └── notifications/ # BullMQ worker
├── websocket/       # WebSocket handlers
└── shared/          # Types, utils, middlewares
```
