# Especificación Técnica de Arquitectura Base v1.1 - Monolito Híbrido Modular

**Versión:** 1.1

**Estado:** Activo

**Última actualización:** 2026-06-29

Este documento define la arquitectura base del proyecto **Himnario MonteSanto Web**. Su propósito es servir como referencia técnica para el desarrollo, mantenimiento, testing, despliegue y evolución del sistema.

La arquitectura queda consolidada como un **Monolito Híbrido Modular** desplegado en **Cloudflare Workers**, combinando una SPA en React con una API backend en Hono dentro de un único artefacto de despliegue.

La versión 1.1 incorpora las decisiones arquitectónicas tomadas sobre identidad, Better Auth, persistencia modular, composición de schemas, cliente HTTP, middleware de autenticación, testing y roadmap.

---

## 1. Objetivos Arquitectónicos

La arquitectura busca cumplir los siguientes objetivos:

- Mantener una separación clara entre dominio, infraestructura backend e infraestructura frontend.
- Permitir despliegue global en Edge mediante Cloudflare Workers.
- Evitar acoplamiento entre React, Hono, Drizzle, Better Auth, Axios y el dominio.
- Facilitar testing unitario, de endpoints y de componentes.
- Permitir evolución modular del sistema sin convertir el código en una estructura monolítica desordenada.
- Centralizar autenticación y autorización en el módulo `identity`.
- Mantener una base compatible con herramientas de automatización y agentes de desarrollo.
- Favorecer refactorizaciones incrementales basadas en necesidades reales.

---

## 2. Principios Arquitectónicos

### 2.1 Monolito Híbrido Modular

El sistema se desarrolla y despliega como un único artefacto, pero internamente se organiza en módulos con límites claros.

Cada módulo agrupa su dominio, backend y frontend.

### 2.2 DDD + Clean Architecture

Se adopta un enfoque de **Domain-Driven Design híbrido** junto con principios de **Clean Architecture**.

El dominio debe mantenerse libre de infraestructura.

El dominio no debe depender de:

- React.
- Hono.
- Drizzle.
- Better Auth.
- Axios.
- Cloudflare Workers.
- Variables de entorno.

### 2.3 Código fuente dentro de `src/`

Todo el código fuente propio de la aplicación reside exclusivamente dentro de:

```text
src/
```

Los artefactos generados por herramientas permanecen fuera de `src` cuando correspondan.

Ejemplos:

```text
drizzle/
dist/
coverage/
.worker-configuration.d.ts
```

Excepción: archivos generados que forman parte explícita del runtime modular, como `auth.schema.ts`, pueden vivir dentro de `src/modules/...` cuando la herramienta lo requiera y la arquitectura lo haya definido así.

---

## 3. Stack Tecnológico Base

### 3.1 Plataforma de Despliegue

- **Servicio:** Cloudflare Workers con Assets/Static Fetching habilitado.
- **Modelo de ejecución:** Edge Computing.
- **Runtime local:** Wrangler.
- **Compatibilidad Node:** `nodejs_compat` habilitado cuando dependencias externas lo requieran, como Better Auth.

### 3.2 Herramientas de Desarrollo, Calidad y Pruebas

- **Gestor de paquetes:** pnpm.
- **Bundler/Dev Server:** Vite.
- **Linter & Formatter:** Biome.
- **Git Hooks:** Husky.
- **Testing:** Vitest.
- **Cloudflare CLI:** Wrangler.

### 3.3 Backend API

- **Framework:** Hono.js.
- **Lenguaje:** TypeScript.
- **Prefijo de API:** `/api/v1/*`.
- **Responsabilidad:** Exponer endpoints HTTP, middlewares, manejo global de errores y servir la SPA en producción.

### 3.4 Frontend SPA

- **Librería:** React.
- **Router:** TanStack Router.
- **Estado servidor/cache:** TanStack Query.
- **Cliente HTTP de negocio:** Axios centralizado.
- **Cliente de autenticación:** cliente oficial de Better Auth.
- **Estilos:** Tailwind CSS.
- **UI:** Shadcn UI.
- **Formularios:** React Hook Form.
- **Validación:** Zod.
- **Tablas complejas:** TanStack Table.

### 3.5 Autenticación y Seguridad

- **Solución:** Better Auth.
- **Persistencia:** Drizzle Adapter.
- **Sesiones:** Cookies seguras `HttpOnly`, `Secure` y `SameSite=Lax`.
- **Módulo responsable:** `identity`.
- **Cliente frontend de autenticación:** cliente oficial de Better Auth.

### 3.6 Base de Datos y Persistencia

- **Motor:** Neon PostgreSQL.
- **Driver:** `@neondatabase/serverless` mediante `drizzle-orm/neon-http`.
- **ORM:** Drizzle ORM.
- **Migraciones:** Drizzle Kit.
- **Conexión:** HTTP mediante `drizzle-orm/neon-http`.

No se utilizan WebSockets para la conexión a la base de datos.

---

## 4. Arquitectura General

### 4.1 Producción

El proyecto se despliega como un único artefacto en Cloudflare Workers.

En producción:

```text
Usuario
  |
  v
Cloudflare Worker
  |-- /api/v1/*  -> Hono API
  |-- /*         -> React SPA estática
```

Cloudflare Workers sirve tanto:

- React.
- API Hono.

Todo vive bajo el mismo dominio.

No existe CORS entre frontend y backend.

Las rutas que comienzan con `/api/v1/*` son resueltas por Hono. El resto de rutas sirven la SPA generada por Vite, permitiendo que TanStack Router controle el enrutamiento del cliente.

### 4.2 Desarrollo Local

En desarrollo se utilizan dos procesos:

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
Vite frontend:    http://localhost:5173
Wrangler API:     http://localhost:8787
```

Vite debe proxyear `/api` hacia Wrangler:

```ts
server: {
  proxy: {
    "/api": {
      target: "http://localhost:8787",
      changeOrigin: true,
    },
  },
}
```

Desde React se consumen APIs con rutas relativas:

```ts
api.get("/health")
```

Nunca se debe hardcodear `http://localhost:8787` en el frontend.

---

## 5. Organización de Módulos

Todos los módulos deben seguir esta estructura base:

```text
module/
├── domain/
├── backend/
└── frontend/
```

Dentro del proyecto:

```text
src/modules/<module>/
  domain/
  backend/
  frontend/
```

### 5.1 Domain

Contiene lógica pura de negocio:

- Entidades.
- Value Objects.
- Reglas de negocio.
- Servicios de dominio.
- Interfaces de repositorios.
- Errores de dominio.
- Modelos propios del dominio o de la aplicación.

Restricciones:

- No importa React.
- No importa Hono.
- No importa Drizzle.
- No importa Better Auth.
- No importa Axios.
- No accede a variables de entorno.

### 5.2 Backend

Contiene infraestructura backend del módulo:

- Rutas Hono.
- Controladores.
- Validaciones de entrada.
- Schemas Drizzle del propio módulo.
- Repositorios Drizzle.
- Adaptadores hacia servicios externos.
- Middlewares propios del módulo.
- Providers de infraestructura.

### 5.3 Frontend

Contiene infraestructura frontend del módulo:

- Componentes React.
- Vistas.
- Hooks.
- Integración con TanStack Query.
- Formularios.
- Adaptadores hacia la API HTTP.
- Clientes frontend específicos permitidos por la arquitectura.

---

## 6. Módulos Transversales

Los módulos transversales definidos actualmente son:

- `identity`.
- `shared`.

### 6.1 `identity`

Centraliza toda la infraestructura relacionada con autenticación y autorización.

Better Auth pertenece a este módulo.

No debe usarse Better Auth directamente fuera de `identity`.

Estructura definida:

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

Responsabilidades:

- Crear y configurar Better Auth.
- Exponer el provider de autenticación.
- Obtener sesiones.
- Mapear usuarios autenticados al modelo de aplicación.
- Proveer middlewares de autenticación y autorización.
- Exponer el cliente oficial de Better Auth en frontend.

### 6.2 `shared`

Contiene infraestructura reutilizable y utilidades compartidas:

```text
src/shared/
  database/
    db.ts
    schema.ts
  http/
    api-client.ts
    api-error.ts
    interceptors.ts
  errors/
  utils/
```

`shared` no debe contener lógica de negocio.

---

## 7. Estructura Base del Proyecto

```text
src/
  index.ts

  config/
    env.ts

  shared/
    database/
      db.ts
      schema.ts
    http/
      api-client.ts
      api-error.ts
      interceptors.ts

  types/
    variables.ts

  modules/
    identity/
      domain/
        models/
          authenticated-user.ts
      backend/
        auth/
          auth.ts
          auth.cli.ts
        providers/
          better-auth.provider.ts
        middleware/
          require-auth.ts
        database/
          schema/
            auth.schema.ts
        mappers/
          authenticated-user.mapper.ts
      frontend/
        auth-client.ts

    <business-module>/
      domain/
      backend/
        database/
          schema/
        routes/
        repositories/
      frontend/
        components/
        hooks/
        pages/
```

---

## 8. Variables de Entorno

Las variables de entorno del Worker se reciben mediante `c.env`.

Wrangler genera los tipos de Cloudflare mediante:

```bash
wrangler types
```

Las variables deben validarse con Zod en:

```text
src/config/env.ts
```

Variables base:

```env
DATABASE_URL=
BETTER_AUTH_URL=
BETTER_AUTH_SECRET=
```

Cloudflare Workers utiliza:

```text
.dev.vars
```

para desarrollo local.

En producción se utilizan Secrets de Wrangler.

Para herramientas Node como Better Auth CLI o Drizzle Kit puede usarse:

```text
.env.local
```

`.dev.vars` y `.env.local` deben estar en `.gitignore`.

No leer `process.env` en runtime Cloudflare Worker.

`process.env` solo puede usarse en scripts CLI o configuración Node, como Drizzle Kit o Better Auth CLI.

---

## 9. Base de Datos

La conexión de runtime se centraliza en:

```text
src/shared/database/db.ts
```

Se utiliza:

- Drizzle ORM.
- Neon PostgreSQL.
- `drizzle-orm/neon-http`.

Ejemplo conceptual:

```ts
import { neon } from "@neondatabase/serverless";
import { drizzle } from "drizzle-orm/neon-http";
import * as schema from "./schema";

export function createDb(databaseUrl: string) {
  const sql = neon(databaseUrl);
  return drizzle(sql, { schema });
}

export type Database = ReturnType<typeof createDb>;
```

Reglas:

- El frontend nunca accede directamente a la base de datos.
- El dominio no importa Drizzle.
- Los schemas Drizzle viven en `backend/database/schema` dentro de cada módulo.
- Las migraciones se generan con Drizzle Kit.
- No se crean conexiones nuevas fuera de `createDb` salvo decisión explícita.
- `db.ts` no debe importar schemas individuales de módulos.

---

## 10. Persistencia Modular

Cada módulo es propietario exclusivamente de sus propios schemas Drizzle.

Ejemplos:

```text
src/modules/identity/backend/database/schema/auth.schema.ts
src/modules/hymns/backend/database/schema/hymn.schema.ts
src/modules/playlists/backend/database/schema/playlist.schema.ts
```

Reglas:

- Ningún módulo debe declarar tablas pertenecientes a otro módulo.
- Las relaciones mediante claves foráneas entre módulos son válidas.
- Un módulo puede referenciar tablas de otro módulo cuando exista una relación real.
- La propiedad de la tabla sigue perteneciendo al módulo que la declara.

Ejemplo:

```text
identity
  auth.schema.ts

hymns
  hymn.schema.ts

playlists
  playlist.schema.ts
```

### 10.1 Composición del Schema Global

La composición del schema global se centraliza en:

```text
src/shared/database/schema.ts
```

Este archivo es el único responsable de componer el schema global utilizado por Drizzle.

Ejemplo conceptual:

```ts
export * from "../../modules/identity/backend/database/schema/auth.schema";
export * from "../../modules/hymns/backend/database/schema/hymn.schema";
export * from "../../modules/playlists/backend/database/schema/playlist.schema";
```

Reglas:

- `shared/database/schema.ts` es el único punto de composición del schema global.
- `db.ts` importa únicamente desde `shared/database/schema.ts`.
- `db.ts` no importa schemas individuales.
- Los módulos no deben componer el schema global.

Configuración esperada de Drizzle Kit:

```ts
schema: "./src/shared/database/schema.ts"
```

---

## 11. Better Auth

Better Auth se considera infraestructura del módulo `identity`.

Archivo runtime:

```text
src/modules/identity/backend/auth/auth.ts
```

Archivo para CLI:

```text
src/modules/identity/backend/auth/auth.cli.ts
```

Schema generado:

```text
src/modules/identity/backend/database/schema/auth.schema.ts
```

### 11.1 `createAuth()`

`createAuth()` actúa como fábrica de Better Auth.

Responsabilidades:

- Crear la instancia de Better Auth.
- Recibir dependencias necesarias del runtime.
- Configurar adapter, secrets, URL base y opciones requeridas.

Reglas:

- Better Auth nunca debe utilizarse directamente fuera del módulo `identity`.
- Toda interacción backend con Better Auth debe encapsularse mediante `BetterAuthProvider`.
- El frontend utilizará exclusivamente el cliente oficial de Better Auth para autenticación.

### 11.2 Tablas de Better Auth

Reglas:

- El schema generado por Better Auth no debe editarse manualmente salvo decisión explícita.
- Las tablas base son `user`, `session`, `account` y `verification`.
- No se debe crear una segunda tabla `users` para representar usuarios de aplicación.
- La tabla `user` de Better Auth es la fuente de verdad de identidad.
- Las tablas de negocio deben referenciar `user.id`.
- `user.id` se trata como `text`, porque Better Auth controla el formato del identificador.
- No agregar lógica de negocio en tablas de Better Auth.

Ejemplo de relación desde una tabla de negocio:

```ts
userId: text("user_id")
  .notNull()
  .references(() => user.id)
```

### 11.3 Ruta de autenticación

Better Auth se monta en Hono bajo:

```text
/api/v1/auth/*
```

---

## 12. BetterAuthProvider

Se introduce `BetterAuthProvider` como adaptador de infraestructura del módulo `identity`.

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
- No representa un servicio de dominio.
- No debe filtrar lógica de Better Auth hacia otros módulos.
- Debe devolver modelos propios o estructuras mapeables hacia modelos propios.

---

## 13. AuthenticatedUser

Se incorpora el modelo `AuthenticatedUser`.

Ubicación esperada:

```text
src/modules/identity/domain/models/authenticated-user.ts
```

Este representa al usuario autenticado dentro de la aplicación.

Reglas:

- No se utilizan directamente los tipos generados por Better Auth fuera del provider.
- Los middlewares y casos de uso deben trabajar con `AuthenticatedUser` o modelos propios de la aplicación.
- El mapeo desde Better Auth hacia `AuthenticatedUser` vive en el módulo `identity`.

Ejemplo conceptual:

```ts
export type AuthenticatedUser = {
  id: string;
  email: string;
  name?: string | null;
};
```

---

## 14. Middleware `requireAuth`

Se implementa `requireAuth` dentro del módulo `identity`.

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
- No expone tipos internos de Better Auth.
- No decide permisos o roles.

La autorización basada en roles corresponde a `requireRole` u otro mecanismo posterior.

---

## 15. RBAC

RBAC se implementa después de validar autenticación básica.

Orden recomendado:

1. Auth base.
2. Login/logout/session.
3. Middleware `requireAuth`.
4. Roles.
5. Middleware `requireRole`.
6. Guards frontend.

Dado que el sistema no contempla organizaciones ni equipos, los roles pueden pertenecer directamente al usuario o ser gestionados mediante el plugin RBAC de Better Auth.

No agregar roles prematuramente en la primera migración de Better Auth salvo decisión explícita.

---

## 16. Arquitectura de Integración HTTP

Toda comunicación entre el frontend y la API de negocio se realiza mediante un cliente HTTP centralizado basado en Axios.

Ubicación:

```text
src/shared/http/api-client.ts
```

Configuración base:

```ts
import axios from "axios";

export const api = axios.create({
  baseURL: "/api/v1",
  withCredentials: true,
});
```

Responsabilidades del cliente Axios:

- Definir la URL base `/api/v1`.
- Enviar cookies cuando corresponda.
- Centralizar interceptores.
- Normalizar errores.
- Agregar logging, trazabilidad o políticas de reintento en el futuro.

Reglas:

- Axios es el cliente HTTP oficial para la API de negocio.
- Ningún módulo debe importar Axios directamente.
- Ningún módulo debe crear instancias propias de Axios.
- Toda llamada a la API de negocio debe pasar por `shared/http/api-client.ts`.
- Las URLs deben ser relativas para funcionar igual en desarrollo y producción.
- TanStack Query debe usar funciones basadas en el cliente Axios centralizado.

---

## 17. Better Auth Client

Las operaciones de autenticación del frontend utilizan exclusivamente el cliente oficial de Better Auth.

Ubicación:

```text
src/modules/identity/frontend/auth-client.ts
```

Better Auth Client se usa para:

- Login.
- Logout.
- Registro.
- Recuperación de contraseña.
- Obtener sesión.

Axios no debe utilizarse para:

- Login.
- Logout.
- Registro.
- Recuperación de contraseña.
- Obtención de sesión.

Axios se usa para:

- APIs de negocio.
- Recursos protegidos del sistema.
- Consultas y mutaciones de módulos funcionales.

---

## 18. Hono y Composición de Aplicación

`src/index.ts` es el punto de composición de la aplicación.

Responsabilidades:

- Crear instancia de Hono.
- Montar rutas.
- Montar middlewares.
- Validar entorno.
- Crear DB por request cuando sea necesario.
- Montar Better Auth.
- Registrar `onError`.
- Servir SPA en producción.

Prohibido:

- Poner reglas de negocio.
- Acceder directamente a entidades de dominio sin pasar por casos de uso cuando corresponda.
- Crear lógica compleja dentro de handlers.

Middlewares esperados:

- `requireAuth` para validar sesión.
- `requireRole` para validar roles.
- Middleware de DB para inyectar `db` en el contexto.
- Middleware global de errores.

---

## 19. Manejo Global de Errores

Se utiliza `app.onError()` en Hono.

Objetivos:

- Evitar fuga de información sensible.
- Responder con JSON consistente.
- Integrarse con TanStack Query y Axios.
- Diferenciar errores de validación, autorización, dominio e infraestructura.

Formato base recomendado:

```json
{
  "success": false,
  "message": "Ha ocurrido un error interno en el servidor del Edge."
}
```

---

## 20. Frontend

### 20.1 TanStack Router

Se usa para rutas tipadas del cliente.

Las rutas protegidas deben usar `beforeLoad` o mecanismos equivalentes para validar sesión antes de renderizar.

### 20.2 TanStack Query

Se usa para:

- Consultas HTTP.
- Mutaciones.
- Caché de servidor.
- Estados de carga/error.

Debe consumir funciones que usen Axios centralizado, no Axios directamente.

### 20.3 Formularios

Los formularios se construyen con:

- React Hook Form.
- Zod.
- Shadcn UI.

---

## 21. Testing

Se adopta Vitest como herramienta única de testing.

Los tests se ubican junto al código que validan.

Ejemplos:

```text
backend/auth/auth-route.test.ts
frontend/login/login.test.tsx
domain/entities/hymn.test.ts
```

No se crea una carpeta global `src/tests`, salvo para pruebas de integración entre múltiples módulos.

### 21.1 Dominio

Tests unitarios puros, sin infraestructura.

### 21.2 Backend

Tests de rutas Hono mediante `.request()`.

Casos mínimos:

- Ruta protegida sin sesión retorna `401`.
- Ruta protegida con sesión válida permite acceso.

### 21.3 Frontend

Tests de componentes con Testing Library y entorno DOM simulado.

### 21.4 Autorización

Cuando RBAC exista, agregar casos mínimos:

- Ruta con rol insuficiente retorna `403`.
- Ruta con rol suficiente permite acceso.

---

## 22. CI/CD

Pipeline recomendado:

1. Instalar dependencias con `pnpm install --frozen-lockfile`.
2. Ejecutar Biome.
3. Ejecutar Vitest.
4. Ejecutar migraciones con Drizzle Kit.
5. Compilar SPA con Vite.
6. Desplegar Worker con Wrangler.

Las migraciones deben ejecutarse antes del despliegue del Worker.

---

## 23. Control de Concurrencia

Para operaciones críticas con riesgo de colisión se utilizará bloqueo pesimista mediante PostgreSQL.

Ejemplo:

```sql
SELECT ... FOR UPDATE
```

Debe reservarse para casos donde la consistencia sea más importante que la concurrencia.

---

## 24. Reglas de Dependencia

Permitido:

```text
frontend -> backend HTTP
backend -> application/domain
backend infrastructure -> Drizzle/Better Auth/Neon
identity backend -> Better Auth
shared/database/schema.ts -> module schemas
```

Prohibido:

```text
domain -> React
domain -> Hono
domain -> Drizzle
domain -> Better Auth
domain -> Axios
frontend -> Drizzle
frontend -> Database
business module -> auth internals
business module -> Better Auth
module -> tables owned by another module
```

---

## 25. Refactorización Evolutiva

La arquitectura debe evolucionar a partir de necesidades reales.

Reglas:

- No introducir abstracciones innecesarias.
- Privilegiar responsabilidades claramente separadas.
- Mantener cambios pequeños y cohesivos.
- Realizar refactorizaciones incrementales al finalizar cada etapa importante.
- Evitar grandes abstracciones anticipadas.

---

## 26. Roadmap Actualizado

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

Alcance:

- Alinear estructura de `identity`.
- Introducir `BetterAuthProvider`.
- Introducir `AuthenticatedUser`.
- Consolidar `requireAuth`.
- Centralizar composición de schema en `shared/database/schema.ts`.
- Asegurar que `db.ts` no importe schemas individuales.
- Confirmar que Axios y Better Auth Client quedan separados.
- Verificar reglas de tests junto al código.

---

## 27. Plan de Implementación Base

### Fase 1: Esqueleto Técnico

- Configurar pnpm, Vite, Biome, Tailwind, Shadcn UI y Vitest.
- Configurar Wrangler.
- Configurar Hono.
- Configurar `wrangler types`.
- Configurar variables de entorno y Zod.
- Configurar Drizzle y Neon.
- Configurar CI/CD base.

### Fase 2: Infraestructura Better Auth

- Instalar Better Auth.
- Crear `createAuth`.
- Crear `auth.cli.ts`.
- Generar `auth.schema.ts`.
- Ejecutar migraciones.
- Crear `BetterAuthProvider`.
- Crear `AuthenticatedUser`.
- Crear mapper hacia `AuthenticatedUser`.

### Fase 3: Integración Better Auth con Hono

- Montar `/api/v1/auth/*`.
- Validar variables requeridas.
- Probar endpoints base de Better Auth.
- Mantener Better Auth encapsulado dentro de `identity`.

### Fase 4: Middleware `requireAuth`

- Implementar `requireAuth`.
- Obtener sesión mediante `BetterAuthProvider`.
- Inyectar `AuthenticatedUser` en contexto Hono.
- Responder `401` sin sesión.
- Agregar tests de ruta protegida.

### Fase 5: Flujo Real de Autenticación

- Probar registro.
- Probar login.
- Probar logout.
- Probar obtención de sesión.
- Validar cookies en desarrollo y producción.

### Fase 6: Login Frontend

- Crear Better Auth Client.
- Crear pantalla de login.
- Configurar guards en TanStack Router.
- Integrar redirección con parámetro `redirect`.

### Fase 7: RBAC para Usuarios Independientes

- Definir roles.
- Configurar Better Auth RBAC o campo extendido.
- Crear `requireRole`.
- Proteger rutas críticas.
- Crear tests de autorización.

### Fase 8: Módulos de Negocio

Cada módulo debe implementarse en orden:

1. Dominio.
2. Backend.
3. Frontend.
4. Tests.

---

## 28. Convenciones Importantes

- No modificar código generado salvo decisión explícita.
- No crear duplicados de tablas gestionadas por Better Auth.
- No acceder a base de datos desde React.
- No importar infraestructura desde dominio.
- No usar Axios directamente fuera de `shared/http`.
- No usar Better Auth Client para APIs de negocio.
- No usar Axios para operaciones propias de Better Auth.
- No usar Better Auth fuera del módulo `identity`.
- No modificar archivos generados como `auth.schema.ts`, migraciones o tipos de Wrangler salvo decisión explícita.
- Cada módulo define únicamente sus propias tablas.
- `shared/database/schema.ts` es el único punto de composición del schema global.
- Preferir rutas relativas en frontend.
- Mantener TypeScript estricto.
- Mantener archivos pequeños y módulos cohesivos.
- Mantener tests junto al código.

---

## 29. Estado del Documento

Este documento representa la **versión 1.1** de la arquitectura del proyecto.

Debe actualizarse cuando se tomen decisiones estructurales relevantes, especialmente sobre:

- Autorización.
- Estructura de módulos.
- Persistencia.
- Integración HTTP.
- Despliegue.
- Testing.
- Refactorizaciones arquitectónicas.

## Historial

### v1.1
- Integración Better Auth.
- BetterAuthProvider.
- AuthenticatedUser.
- shared/database/schema.ts.
- Axios.
- Arquitectura evolutiva.

### v1.0
- Arquitectura inicial.
