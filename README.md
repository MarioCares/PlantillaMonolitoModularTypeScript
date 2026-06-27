# Especificación Técnica de Arquitectura Base - Monolito Híbrido Modular

Este documento define la arquitectura base del proyecto **Himnario MonteSanto Web**. Su propósito es servir como referencia técnica para el desarrollo, mantenimiento, testing, despliegue y evolución del sistema.

La arquitectura está diseñada como un **monolito híbrido modular** desplegado en **Cloudflare Workers**, combinando una SPA en React con una API backend en Hono dentro de un único artefacto de despliegue.

---

## 1. Objetivos Arquitectónicos

La arquitectura busca cumplir los siguientes objetivos:

- Mantener una separación clara entre dominio, infraestructura backend e infraestructura frontend.
- Permitir despliegue global en Edge mediante Cloudflare Workers.
- Evitar acoplamiento entre React, Hono, Drizzle, Better Auth y el dominio.
- Facilitar testing unitario, de endpoints y de componentes.
- Permitir evolución modular del sistema sin convertir el código en una estructura monolítica desordenada.
- Mantener una base compatible con herramientas de automatización y agentes de desarrollo.

---

## 2. Stack Tecnológico Base

### Plataforma de Despliegue

- **Servicio:** Cloudflare Workers con Assets/Static Fetching habilitado.
- **Modelo de ejecución:** Edge Computing.
- **Runtime local:** Wrangler.
- **Compatibilidad Node:** `nodejs_compat` habilitado cuando dependencias externas lo requieran, como Better Auth.

### Herramientas de Desarrollo, Calidad y Pruebas

- **Gestor de paquetes:** pnpm.
- **Bundler/Dev Server:** Vite.
- **Linter & Formatter:** Biome.
- **Git Hooks:** Husky.
- **Testing:** Vitest.
- **Cloudflare CLI:** Wrangler.

### Backend API

- **Framework:** Hono.js.
- **Lenguaje:** TypeScript.
- **Prefijo de API:** `/api/v1/*`.
- **Responsabilidad:** Exponer endpoints HTTP, middlewares, manejo global de errores y servir la SPA en producción.

### Frontend SPA

- **Librería:** React.
- **Router:** TanStack Router.
- **Estado servidor/cache:** TanStack Query.
- **Cliente HTTP de negocio:** Axios.
- **Estilos:** Tailwind CSS.
- **UI:** Shadcn UI.
- **Formularios:** React Hook Form.
- **Validación:** Zod.
- **Tablas complejas:** TanStack Table.

### Autenticación y Seguridad

- **Solución:** Better Auth.
- **Persistencia:** Drizzle Adapter.
- **Sesiones:** Cookies seguras `HttpOnly`, `Secure` y `SameSite=Lax`.
- **Módulo responsable:** `identity`.
- **Cliente frontend de autenticación:** cliente oficial de Better Auth.

### Base de Datos y Persistencia

- **Motor:** Neon PostgreSQL.
- **Driver:** `@neondatabase/serverless` mediante `drizzle-orm/neon-http`.
- **ORM:** Drizzle ORM.
- **Migraciones:** Drizzle Kit.

---

## 3. Arquitectura General

### Monolito Híbrido Modular

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

Las rutas que comienzan con `/api/v1/*` son resueltas por Hono. El resto de rutas sirven la SPA generada por Vite, permitiendo que TanStack Router controle el enrutamiento del cliente.

### Desarrollo Local

En desarrollo se utilizan dos procesos:

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

## 4. DDD Híbrido y Clean Architecture

Se adopta un enfoque de **Domain-Driven Design híbrido** con módulos verticales. Cada módulo de negocio debe agrupar sus responsabilidades en capas internas.

Estructura base de un módulo de negocio:

```text
src/modules/<module>/
  domain/
  backend/
  frontend/
```

### Domain

Contiene lógica pura de negocio:

- Entidades.
- Value Objects.
- Reglas de negocio.
- Servicios de dominio.
- Interfaces de repositorios.
- Errores de dominio.

Restricciones:

- No importa React.
- No importa Hono.
- No importa Drizzle.
- No importa Better Auth.
- No accede a variables de entorno.

### Backend

Contiene infraestructura backend del módulo:

- Rutas Hono.
- Controladores.
- Validaciones de entrada.
- Schemas Drizzle.
- Repositorios Drizzle.
- Adaptadores hacia servicios externos.

### Frontend

Contiene infraestructura frontend del módulo:

- Componentes React.
- Vistas.
- Hooks.
- Integración con TanStack Query.
- Formularios.
- Adaptadores hacia la API HTTP.

---

## 5. Módulos Transversales

No todos los módulos representan dominio de negocio. Algunos son transversales.

### `identity`

Responsable de autenticación, sesiones y autorización.

Ubicación:

```text
src/modules/identity/
  backend/
    auth/
    database/
      schema/
  frontend/
```

Better Auth pertenece a infraestructura dentro del módulo `identity`.

El dominio de negocio no debe depender de Better Auth.

### `shared`

Contiene infraestructura reutilizable y utilidades compartidas:

```text
src/shared/
  database/
  http/
  errors/
  utils/
```

`shared` no debe convertirse en un basurero de lógica de negocio.

### `config`

Contiene validación y lectura de configuración del runtime:

```text
src/config/env.ts
```

### `types`

Contiene tipos de composición de runtime, como variables del contexto Hono:

```text
src/types/variables.ts
```

---

## 6. Estructura Base del Proyecto

```text
src/
  index.ts

  config/
    env.ts

  shared/
    database/
      db.ts
    http/
      api-client.ts
      api-error.ts
      interceptors.ts

  types/
    variables.ts

  modules/
    identity/
      backend/
        auth/
          auth.ts
          auth.cli.ts
        database/
          schema/
            auth.schema.ts
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

## 7. Variables de Entorno

Las variables de entorno del Worker se reciben mediante `c.env`.

Wrangler genera los tipos de Cloudflare en:

```text
worker-configuration.d.ts
```

Comando:

```bash
pnpm wrangler types
```

Las variables deben validarse con Zod en `src/config/env.ts`.

Variables base:

```env
DATABASE_URL=
BETTER_AUTH_URL=
BETTER_AUTH_SECRET=
```

En desarrollo local para Workers se usa:

```text
.dev.vars
```

Para herramientas Node como Better Auth CLI o Drizzle Kit puede usarse:

```text
.env.local
```

Ambos deben estar en `.gitignore`.

---

## 8. Base de Datos

La conexión de runtime se centraliza en:

```text
src/shared/database/db.ts
```

Ejemplo conceptual:

```ts
import { neon } from "@neondatabase/serverless";
import { drizzle } from "drizzle-orm/neon-http";

export function createDb(databaseUrl: string) {
  const sql = neon(databaseUrl);
  return drizzle(sql);
}

export type Database = ReturnType<typeof createDb>;
```

Reglas:

- El frontend nunca accede directamente a la base de datos.
- El dominio no importa Drizzle.
- Los schemas Drizzle viven en `backend/database/schema`.
- Las migraciones se generan con Drizzle Kit.

Configuración esperada de Drizzle:

```ts
schema: "./src/modules/**/backend/database/schema/*.schema.ts"
```

---

## 9. Better Auth

Better Auth se configura dentro de `identity/backend/auth`.

Archivo runtime:

```text
src/modules/identity/backend/auth/auth.ts
```

Archivo para CLI:

```text
src/modules/identity/backend/auth/auth.cli.ts
```

Schemas generados:

```text
src/modules/identity/backend/database/schema/auth.schema.ts
```

Reglas:

- El schema generado por Better Auth no debe editarse manualmente salvo decisión explícita.
- Las tablas base son `user`, `session`, `account` y `verification`.
- No se debe crear una segunda tabla `users` para representar usuarios de aplicación.
- Las tablas de negocio deben referenciar `user.id`.
- `user.id` se trata como `text`, porque Better Auth controla el formato del identificador.

Ejemplo de relación desde una tabla de negocio:

```ts
userId: text("user_id")
  .notNull()
  .references(() => user.id)
```

### Ruta de autenticación

Better Auth se monta en Hono bajo:

```text
/api/v1/auth/*
```

### RBAC

RBAC no se agrega en la primera migración de autenticación.

Orden recomendado:

1. Generar schemas base.
2. Migrar tablas base.
3. Probar registro, login, logout y sesión.
4. Crear middleware `requireAuth`.
5. Agregar RBAC.
6. Crear middleware `requireRole`.

Dado que el sistema no contempla organizaciones ni equipos, los roles pueden pertenecer directamente al usuario o ser gestionados mediante el plugin RBAC de Better Auth.

---

## 10. Arquitectura de Integración HTTP

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

- Ningún módulo debe importar Axios directamente.
- Toda llamada a la API de negocio debe pasar por `shared/http/api-client.ts`.
- Las URLs deben ser relativas para funcionar igual en desarrollo y producción.
- TanStack Query debe usar funciones basadas en el cliente Axios centralizado.

### Separación entre Axios y Better Auth Client

Las operaciones de autenticación no deben implementarse con Axios directamente.

Se debe usar el cliente oficial de Better Auth en:

```text
src/modules/identity/frontend/auth-client.ts
```

Better Auth Client se usa para:

- Login.
- Logout.
- Registro.
- Recuperación de contraseña.
- Obtener sesión.

Axios se usa para:

- APIs de negocio.
- Recursos protegidos del sistema.
- Consultas y mutaciones de módulos funcionales.

---

## 11. Hono y Middlewares

`src/index.ts` es el punto de composición de la aplicación.

Responsabilidades:

- Crear instancia de Hono.
- Montar rutas.
- Validar entorno.
- Crear DB por request cuando sea necesario.
- Montar Better Auth.
- Manejar errores globales.

No debe contener lógica de negocio.

Middlewares esperados:

- `requireAuth` para validar sesión.
- `requireRole` para validar roles.
- Middleware de DB para inyectar `db` en el contexto.
- Middleware global de errores.

---

## 12. Manejo Global de Errores

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

## 13. Frontend

### TanStack Router

Se usa para rutas tipadas del cliente.

Las rutas protegidas deben usar `beforeLoad` o mecanismos equivalentes para validar sesión antes de renderizar.

### TanStack Query

Se usa para:

- Consultas HTTP.
- Mutaciones.
- Caché de servidor.
- Estados de carga/error.

Debe consumir funciones que usen Axios centralizado, no Axios directamente.

### Formularios

Los formularios se construyen con:

- React Hook Form.
- Zod.
- Shadcn UI.

---

## 14. Testing

Se adopta Vitest como herramienta única de testing.

### Dominio

Tests unitarios puros, sin infraestructura.

### Backend

Tests de rutas Hono mediante `.request()`.

### Frontend

Tests de componentes con Testing Library y entorno DOM simulado.

### Autenticación

Casos mínimos:

- Ruta protegida sin sesión retorna `401`.
- Ruta protegida con sesión válida permite acceso.
- Ruta con rol insuficiente retorna `403`.
- Ruta con rol suficiente permite acceso.

---

## 15. CI/CD

Pipeline recomendado:

1. Instalar dependencias con `pnpm install --frozen-lockfile`.
2. Ejecutar Biome.
3. Ejecutar Vitest.
4. Ejecutar migraciones con Drizzle Kit.
5. Compilar SPA con Vite.
6. Desplegar Worker con Wrangler.

Las migraciones deben ejecutarse antes del despliegue del Worker.

---

## 16. Control de Concurrencia

Para operaciones críticas con riesgo de colisión se utilizará bloqueo pesimista mediante PostgreSQL.

Ejemplo:

```sql
SELECT ... FOR UPDATE
```

Debe reservarse para casos donde la consistencia sea más importante que la concurrencia.

---

## 17. Reglas de Dependencia

Permitido:

```text
frontend -> backend HTTP
backend -> application/domain
backend infrastructure -> Drizzle/Better Auth/Neon
```

Prohibido:

```text
domain -> React
domain -> Hono
domain -> Drizzle
domain -> Better Auth
frontend -> Drizzle
frontend -> Database
business module -> auth internals
```

---

## 18. Plan de Implementación Base

### Fase 1: Esqueleto Técnico

- Configurar pnpm, Vite, Biome, Tailwind, Shadcn UI y Vitest.
- Configurar Wrangler.
- Configurar Hono.
- Configurar `wrangler types`.
- Configurar variables de entorno y Zod.
- Configurar Drizzle y Neon.
- Configurar CI/CD base.

### Fase 2: Identidad y Autenticación

- Instalar Better Auth.
- Crear `createAuth`.
- Crear `auth.cli.ts`.
- Generar `auth.schema.ts`.
- Ejecutar migraciones.
- Montar `/api/v1/auth/*`.
- Probar login, registro, logout y sesión.
- Crear `requireAuth`.

### Fase 3: RBAC

- Definir roles.
- Configurar Better Auth RBAC o campo extendido.
- Crear `requireRole`.
- Proteger rutas críticas.
- Crear tests de autorización.

### Fase 4: Frontend de Autenticación

- Crear Better Auth Client.
- Crear pantalla de login.
- Configurar guards en TanStack Router.
- Integrar redirección con parámetro `redirect`.

### Fase 5: Módulos de Negocio

Cada módulo debe implementarse en orden:

1. Dominio.
2. Backend.
3. Frontend.
4. Tests.

---

## 19. Convenciones Importantes

- No modificar código generado salvo decisión explícita.
- No crear duplicados de tablas gestionadas por Better Auth.
- No acceder a base de datos desde React.
- No importar infraestructura desde dominio.
- No usar Axios directamente fuera de `shared/http`.
- No usar Better Auth Client para APIs de negocio.
- No usar Axios para operaciones propias de Better Auth.
- Preferir rutas relativas en frontend.
- Mantener TypeScript estricto.
- Mantener archivos pequeños y módulos cohesivos.

---

## 20. Estado del Documento

Este documento representa la versión base de arquitectura del proyecto.

Debe actualizarse cuando se tomen decisiones estructurales relevantes, especialmente sobre:

- Autorización.
- Estructura de módulos.
- Persistencia.
- Integración HTTP.
- Despliegue.
- Testing.
