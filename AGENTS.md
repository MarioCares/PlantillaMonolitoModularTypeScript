# AGENTS.md

**Versión:** 1.1

**Estado:** Activo

**Última actualización:** 2026-06-29

Este archivo define las reglas que deben seguir los agentes de desarrollo, asistentes de código y herramientas automatizadas al trabajar en este repositorio.

Antes de modificar código, todo agente debe leer este archivo y `ARCHITECTURE.md`.

---

## 1. Principio General

Este proyecto usa una arquitectura de **Monolito Híbrido Modular** con **DDD + Clean Architecture**.

El objetivo no es solo hacer que el código funcione. El objetivo es mantener una estructura clara, mantenible y consistente.

La arquitectura debe evolucionar a partir de necesidades reales. No introducir abstracciones innecesarias.

---

## 2. Regla Principal

No romper las dependencias arquitectónicas.

El dominio es puro.

Nunca importar en `domain`:

- React.
- Hono.
- Drizzle.
- Better Auth.
- Axios.
- Cloudflare Workers.
- Variables de entorno.

---

## 3. Código Fuente

El código fuente de la aplicación reside exclusivamente dentro de:

```text
src/
```

Los artefactos generados por herramientas deben permanecer fuera de `src` cuando correspondan.

Ejemplos:

```text
drizzle/
dist/
coverage/
```

Excepción: archivos generados que forman parte explícita del runtime modular, como `auth.schema.ts`, pueden vivir dentro de `src/modules/...` cuando la arquitectura lo defina.

---

## 4. Estructura de Módulos

Cada módulo debe seguir esta estructura:

```text
src/modules/<module>/
  domain/
  backend/
  frontend/
```

No crear carpetas improvisadas si ya existe una convención definida.

Cada módulo es responsable de mantener sus límites internos.

---

## 5. Módulos Transversales

Los módulos transversales definidos actualmente son:

- `identity`.
- `shared`.

### `identity`

Centraliza autenticación, sesiones y autorización.

Better Auth vive aquí.

No crear otro módulo `auth` salvo decisión explícita.

Estructura esperada:

```text
src/modules/identity/
  domain/
    models/
      authenticated-user.ts

  backend/
    auth/
    providers/
    middleware/
    database/
    mappers/

  frontend/
    auth-client.ts
```

### `shared`

Contiene infraestructura reutilizable.

No colocar lógica de negocio en `shared`.

Ubicaciones relevantes:

```text
src/shared/database/db.ts
src/shared/database/schema.ts
src/shared/http/api-client.ts
```

---

## 6. Better Auth

Reglas:

- Better Auth pertenece a `src/modules/identity`.
- Better Auth se considera infraestructura del módulo `identity`.
- `createAuth()` actúa como fábrica de Better Auth.
- Better Auth nunca debe utilizarse directamente fuera del módulo `identity`.
- Toda interacción backend con Better Auth debe encapsularse mediante `BetterAuthProvider`.
- El schema generado debe vivir en `src/modules/identity/backend/database/schema/auth.schema.ts`.
- No modificar manualmente el schema generado salvo instrucción explícita.
- No crear una tabla adicional `users`.
- La tabla `user` de Better Auth es la fuente de verdad de identidad.
- Las tablas de negocio deben referenciar `user.id` como `text`.
- No agregar lógica de negocio en tablas de Better Auth.

---

## 7. BetterAuthProvider

`BetterAuthProvider` es un adaptador de infraestructura.

Ubicación esperada:

```text
src/modules/identity/backend/providers/better-auth.provider.ts
```

Responsabilidades:

- Obtener la sesión actual.
- Encapsular llamadas al SDK de Better Auth.
- Evitar dependencias directas de Better Auth desde middlewares.
- Evitar dependencias directas de Better Auth desde futuros casos de uso.

Reglas:

- No contiene reglas de negocio.
- No es un servicio de dominio.
- No debe exponer tipos internos de Better Auth fuera del módulo `identity`.

---

## 8. AuthenticatedUser

El modelo `AuthenticatedUser` representa al usuario autenticado dentro de la aplicación.

Ubicación esperada:

```text
src/modules/identity/domain/models/authenticated-user.ts
```

Reglas:

- No usar directamente tipos generados por Better Auth fuera del provider.
- Mapear la sesión de Better Auth hacia `AuthenticatedUser` dentro de `identity`.
- Inyectar `AuthenticatedUser` en el contexto de Hono cuando exista sesión válida.

---

## 9. Middleware `requireAuth`

`requireAuth` vive dentro del módulo `identity`.

Ubicación esperada:

```text
src/modules/identity/backend/middleware/require-auth.ts
```

Responsabilidades:

- Obtener la sesión mediante `BetterAuthProvider`.
- Mapear el usuario autenticado.
- Inyectar `AuthenticatedUser` en el contexto de Hono.
- Responder `401` cuando no exista sesión.

Reglas:

- No contiene lógica de negocio.
- No llama directamente al SDK de Better Auth.
- No decide permisos ni roles.

---

## 10. RBAC

RBAC se implementa después de validar autenticación básica.

Orden correcto:

1. Auth base.
2. Login/logout/session.
3. Middleware `requireAuth`.
4. Roles.
5. Middleware `requireRole`.
6. Guards frontend.

No agregar roles prematuramente en la primera migración de Better Auth salvo instrucción explícita.

---

## 11. Base de Datos

La conexión a la base se centraliza en:

```text
src/shared/database/db.ts
```

Se utiliza:

- Drizzle ORM.
- Neon PostgreSQL.
- `drizzle-orm/neon-http`.

No se utilizan WebSockets para la conexión a la base de datos.

Reglas:

- No crear conexiones nuevas fuera de `createDb` salvo instrucción explícita.
- No acceder a la base desde React.
- No importar Drizzle en el dominio.
- `db.ts` no debe importar schemas individuales de módulos.
- `db.ts` debe consumir el schema global desde `shared/database/schema.ts`.

---

## 12. Schemas Drizzle por Módulo

Cada módulo es propietario exclusivamente de sus propios schemas Drizzle.

Ubicación esperada:

```text
src/modules/<module>/backend/database/schema/*.schema.ts
```

Ejemplos:

```text
src/modules/identity/backend/database/schema/auth.schema.ts
src/modules/hymns/backend/database/schema/hymn.schema.ts
src/modules/playlists/backend/database/schema/playlist.schema.ts
```

Reglas:

- Cada módulo define únicamente sus propias tablas.
- Ningún módulo debe declarar tablas pertenecientes a otro módulo.
- Las relaciones mediante claves foráneas entre módulos son válidas.
- La propiedad de una tabla pertenece al módulo que la declara.

---

## 13. Composición del Schema Global

La composición del schema global se centraliza en:

```text
src/shared/database/schema.ts
```

Este archivo es el único responsable de componer el schema global utilizado por Drizzle.

Reglas:

- `shared/database/schema.ts` exporta los schemas de los módulos.
- `db.ts` importa únicamente desde `shared/database/schema.ts`.
- `db.ts` no importa schemas individuales.
- Los módulos no componen el schema global.

Configuración esperada de Drizzle Kit:

```ts
schema: "./src/shared/database/schema.ts"
```

---

## 14. Hono

`src/index.ts` es composición de aplicación, no dominio.

Permitido:

- Montar rutas.
- Montar middlewares.
- Validar entorno.
- Registrar `onError`.
- Integrar Better Auth.
- Servir la SPA en producción.

Prohibido:

- Poner reglas de negocio.
- Acceder directamente a entidades de dominio sin pasar por casos de uso cuando corresponda.
- Crear lógica compleja dentro de handlers.

---

## 15. Variables de Entorno

Las variables del Worker se leen desde `c.env`.

Deben validarse con Zod en:

```text
src/config/env.ts
```

Cloudflare Workers utiliza:

```text
.dev.vars
```

para desarrollo local.

Producción utiliza Secrets de Wrangler.

Los tipos de `Env` son generados mediante:

```bash
wrangler types
```

No leer `process.env` en runtime Cloudflare Worker.

`process.env` solo puede usarse en scripts CLI o configuración Node, como Drizzle Kit o Better Auth CLI.

---

## 16. Axios

Axios se usa exclusivamente para consumir API de negocio desde frontend.

La instancia central debe vivir en:

```text
src/shared/http/api-client.ts
```

Reglas:

- No importar Axios directamente desde módulos.
- No crear múltiples instancias de Axios.
- Usar `baseURL: "/api/v1"`.
- Usar `withCredentials: true` cuando corresponda.
- Centralizar interceptores y manejo de errores.

---

## 17. Better Auth Client

Las operaciones de autenticación del frontend deben usar exclusivamente el cliente oficial de Better Auth.

Ubicación:

```text
src/modules/identity/frontend/auth-client.ts
```

Usar Better Auth Client para:

- Login.
- Logout.
- Registro.
- Recuperación de contraseña.
- Obtener sesión.

No usar Axios para:

- Login.
- Logout.
- Registro.
- Recuperación de contraseña.
- Obtención de sesión.

---

## 18. TanStack Query

Los hooks de TanStack Query deben consumir funciones de API, no llamar HTTP directamente dentro del componente.

Preferido:

```ts
useQuery({
  queryKey: ["hymns"],
  queryFn: getHymns,
});
```

Evitar:

```ts
useQuery({
  queryKey: ["hymns"],
  queryFn: () => axios.get(...),
});
```

---

## 19. Frontend

Reglas:

- No acceder a la base de datos.
- No importar Drizzle.
- No importar código backend.
- No duplicar reglas de negocio críticas.
- Usar Zod para validación de formularios.
- Usar React Hook Form para formularios.
- Usar Shadcn UI para componentes base.
- Usar Axios centralizado solo para API de negocio.
- Usar Better Auth Client para autenticación.

---

## 20. Dominio

El dominio debe ser testeable sin infraestructura.

Preferir:

- Entidades puras.
- Value Objects.
- Funciones puras.
- Interfaces de repositorio.
- Errores explícitos de dominio.
- Modelos propios de aplicación.

Evitar:

- Acoplar a frameworks.
- Leer variables de entorno.
- Usar fechas globales sin inyección cuando afecte reglas críticas.
- Usar base de datos directamente.
- Usar tipos de Better Auth directamente.

---

## 21. Testing

Cuando se modifique lógica de dominio, agregar o actualizar tests unitarios.

Cuando se modifiquen rutas Hono, agregar o actualizar tests con `.request()`.

Cuando se modifique UI funcional, agregar o actualizar tests de componentes cuando corresponda.

Las pruebas deben ubicarse junto al código que validan.

No crear una carpeta global `src/tests` salvo para pruebas de integración que involucren múltiples módulos.

Ejemplos:

```text
modules/identity/backend/auth/auth-route.test.ts
modules/identity/backend/middleware/require-auth.test.ts
modules/hymns/domain/hymn.test.ts
modules/hymns/frontend/hymn-page.test.tsx
```

Comandos esperados:

```bash
pnpm vitest run
pnpm biome check .
```

---

## 22. Biome

Biome es la fuente de verdad para linting y formato.

No introducir ESLint o Prettier salvo decisión explícita.

Antes de finalizar cambios, ejecutar Biome o asegurarse de que el código cumple sus reglas.

---

## 23. Migraciones

Reglas:

- Las migraciones se generan con Drizzle Kit.
- No editar migraciones aplicadas salvo que el proyecto esté en etapa inicial y se indique explícitamente.
- No borrar migraciones ya aplicadas en ambientes compartidos.
- Revisar SQL generado antes de aplicarlo.
- No modificar migraciones generadas salvo instrucción explícita.

---

## 24. Cloudflare Workers

Reglas:

- Usar Wrangler para probar backend local.
- `.dev.vars` solo lo carga Wrangler.
- Vite no carga `c.env`.
- Para probar frontend + backend en desarrollo, usar Vite con proxy hacia Wrangler.
- Si una dependencia requiere APIs Node, usar `nodejs_compat`.
- Producción usa Secrets de Wrangler.

---

## 25. Desarrollo Local

Patrón esperado:

```text
React
  ↓
Vite (5173)
  ↓
Proxy
  ↓
Wrangler (8787)
  ↓
Hono
```

Representación con puertos:

```text
Vite frontend -> localhost:5173
Wrangler API  -> localhost:8787
```

El frontend debe consumir rutas relativas:

```ts
api.get("/health")
```

No hardcodear hosts locales en código fuente.

---

## 26. Producción

En producción todo vive bajo el mismo origen:

```text
https://dominio.com/
https://dominio.com/api/v1/*
```

Cloudflare Workers sirve tanto:

- React.
- API Hono.

No agregar CORS innecesario salvo que exista un consumidor externo real.

---

## 27. Convenciones de Nombres

- Módulos en minúscula: `identity`, `hymns`, `favorites`.
- Schemas Drizzle: `*.schema.ts`.
- Repositorios infraestructura: `*.repository.ts`.
- Hooks React: `use-*.ts` o `use-*.tsx`.
- Clientes externos: `*-client.ts`.
- Middlewares: `require-auth.ts`, `require-role.ts`.
- Providers: `*.provider.ts`.
- Mappers: `*.mapper.ts`.

---

## 28. Código Generado

Código generado por herramientas externas debe tratarse como propiedad de la herramienta.

Ejemplos:

- `auth.schema.ts` generado por Better Auth.
- Migraciones generadas por Drizzle Kit.
- Archivos generados por TanStack Router.
- Tipos generados por Wrangler.

No modificar manualmente salvo instrucción explícita.

---

## 29. Antes de Crear Código Nuevo

Antes de agregar una funcionalidad:

1. Identificar el módulo correcto.
2. Determinar si es dominio, backend o frontend.
3. Revisar si ya existe una abstracción compartida.
4. Evitar duplicar infraestructura.
5. Mantener el cambio pequeño y cohesivo.
6. Verificar que no se esté importando Better Auth fuera de `identity`.
7. Verificar que no se esté importando Axios directamente.
8. Verificar que el módulo no esté declarando tablas de otro módulo.

---

## 30. Antes de Finalizar una Tarea

Verificar:

- El código compila.
- No se rompieron reglas de dependencia.
- No se modificó código generado innecesariamente.
- Biome no reporta errores.
- Los tests relevantes pasan.
- La estructura respeta `ARCHITECTURE.md`.
- Los tests están junto al código que validan.
- `shared/database/schema.ts` sigue siendo el único punto de composición del schema global.

---

## 31. Roadmap Actualizado

Las issues quedan reorganizadas como:

1. Infraestructura Better Auth.
2. Integración Better Auth con Hono.
3. Middleware `requireAuth`.
4. Flujo real de autenticación: registro, login y sesión.
5. Login frontend.
6. RBAC para usuarios independientes.

Además, se incorpora:

### AR-01: Architectural Refactoring

Objetivo:

Consolidar la infraestructura antes de comenzar el desarrollo del frontend.

Alcance esperado:

- Alinear estructura de `identity`.
- Introducir `BetterAuthProvider`.
- Introducir `AuthenticatedUser`.
- Consolidar `requireAuth`.
- Centralizar composición de schema en `shared/database/schema.ts`.
- Asegurar que `db.ts` no importe schemas individuales.
- Confirmar separación entre Axios y Better Auth Client.
- Verificar tests junto al código.

---

## 32. Reglas Nuevas para Agentes

- No utilizar Better Auth fuera del módulo `identity`.
- No importar Axios directamente.
- No modificar archivos generados como `auth.schema.ts`, migraciones o tipos generados.
- Mantener los tests junto al código.
- Cada módulo define únicamente sus propias tablas.
- `shared/database/schema.ts` es el único punto de composición del schema global.
- Mantener el dominio libre de dependencias hacia infraestructura.
- Favorecer pequeños refactorings evolutivos antes que grandes abstracciones anticipadas.

---

## 33. Regla de Oro

Si una tarea requiere romper una regla de arquitectura, no lo hagas silenciosamente.

Primero documenta la razón y solicita confirmación.

## Historial de revisiones

### v1.1

Nuevas reglas incorporadas:

- Better Auth únicamente puede utilizarse desde el módulo Identity.
- BetterAuthProvider es el único punto de acceso al SDK.
- Axios se utiliza exclusivamente para la API de negocio.
- Better Auth Client se utiliza exclusivamente para autenticación.
- shared/database/schema.ts compone el schema global.
- Los tests deben permanecer junto al código que validan.

### v1.0

Versión inicial de reglas para agentes.
