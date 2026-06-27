# AGENTS.md

Este archivo define las reglas que deben seguir los agentes de desarrollo, asistentes de código y herramientas automatizadas al trabajar en este repositorio.

Antes de modificar código, todo agente debe leer este archivo y `ARCHITECTURE.md`.

---

## 1. Principio General

Este proyecto usa una arquitectura de **monolito híbrido modular** con **DDD + Clean Architecture**.

El objetivo no es solo hacer que el código funcione. El objetivo es mantener una estructura clara, mantenible y consistente.

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

## 3. Estructura de Módulos

Cada módulo de negocio debe seguir esta estructura:

```text
src/modules/<module>/
  domain/
  backend/
  frontend/
```

No crear carpetas improvisadas si ya existe una convención definida.

---

## 4. Módulos Transversales

### `identity`

Contiene autenticación, sesiones y autorización.

Better Auth vive aquí.

No crear otro módulo `auth` salvo decisión explícita.

### `shared`

Contiene infraestructura reutilizable.

No colocar lógica de negocio en `shared`.

### `config`

Contiene validación de configuración y entorno.

### `types`

Contiene tipos de composición de runtime, como variables de Hono.

---

## 5. Better Auth

Reglas:

- Better Auth pertenece a `src/modules/identity`.
- El schema generado debe vivir en `src/modules/identity/backend/database/schema/auth.schema.ts`.
- No modificar manualmente el schema generado salvo instrucción explícita.
- No crear una tabla adicional `users`.
- La tabla `user` de Better Auth es la fuente de verdad de identidad.
- Las tablas de negocio deben referenciar `user.id` como `text`.
- No agregar lógica de negocio en tablas de Better Auth.

---

## 6. RBAC

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

## 7. Base de Datos

La conexión a la base se centraliza en:

```text
src/shared/database/db.ts
```

Reglas:

- No crear conexiones nuevas fuera de `createDb` salvo instrucción explícita.
- No acceder a la base desde React.
- No importar Drizzle en el dominio.
- Los schemas Drizzle deben vivir en:

```text
src/modules/<module>/backend/database/schema/*.schema.ts
```

---

## 8. Hono

`src/index.ts` es composición de aplicación, no dominio.

Permitido:

- Montar rutas.
- Montar middlewares.
- Validar entorno.
- Registrar `onError`.
- Integrar Better Auth.

Prohibido:

- Poner reglas de negocio.
- Acceder directamente a entidades de dominio sin pasar por casos de uso.
- Crear lógica compleja dentro de handlers.

---

## 9. Variables de Entorno

Las variables del Worker se leen desde `c.env`.

Deben validarse con Zod en:

```text
src/config/env.ts
```

No leer `process.env` en runtime Cloudflare Worker.

`process.env` solo puede usarse en scripts CLI o configuración Node, como Drizzle Kit o Better Auth CLI.

---

## 10. Axios

Axios se usa exclusivamente para consumir API de negocio desde frontend.

La instancia central debe vivir en:

```text
src/shared/http/api-client.ts
```

Reglas:

- No importar Axios directamente desde módulos.
- No crear múltiples instancias de Axios sin justificación.
- Usar `baseURL: "/api/v1"`.
- Usar `withCredentials: true` cuando corresponda.
- Centralizar interceptores y manejo de errores.

---

## 11. Better Auth Client

Las operaciones de autenticación del frontend deben usar el cliente oficial de Better Auth.

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

No usar Axios directamente para estas operaciones salvo instrucción explícita.

---

## 12. TanStack Query

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

## 13. Frontend

Reglas:

- No acceder a la base de datos.
- No importar Drizzle.
- No importar código backend.
- No duplicar reglas de negocio críticas.
- Usar Zod para validación de formularios.
- Usar React Hook Form para formularios.
- Usar Shadcn UI para componentes base.

---

## 14. Dominio

El dominio debe ser testeable sin infraestructura.

Preferir:

- Entidades puras.
- Value Objects.
- Funciones puras.
- Interfaces de repositorio.
- Errores explícitos de dominio.

Evitar:

- Acoplar a frameworks.
- Leer variables de entorno.
- Usar fechas globales sin inyección cuando afecte reglas críticas.
- Usar base de datos directamente.

---

## 15. Testing

Cuando se modifique lógica de dominio, agregar o actualizar tests unitarios.

Cuando se modifiquen rutas Hono, agregar o actualizar tests con `.request()`.

Cuando se modifique UI funcional, agregar o actualizar tests de componentes cuando corresponda.

Comandos esperados:

```bash
pnpm vitest run
pnpm biome check .
```

---

## 16. Biome

Biome es la fuente de verdad para linting y formato.

No introducir ESLint o Prettier salvo decisión explícita.

Antes de finalizar cambios, ejecutar Biome o asegurarse de que el código cumple sus reglas.

---

## 17. Migraciones

Reglas:

- Las migraciones se generan con Drizzle Kit.
- No editar migraciones aplicadas salvo que el proyecto esté en etapa inicial y se indique explícitamente.
- No borrar migraciones ya aplicadas en ambientes compartidos.
- Revisar SQL generado antes de aplicarlo.

---

## 18. Cloudflare Workers

Reglas:

- Usar Wrangler para probar backend local.
- `.dev.vars` solo lo carga Wrangler.
- Vite no carga `c.env`.
- Para probar frontend + backend en desarrollo, usar Vite con proxy hacia Wrangler.
- Si una dependencia requiere APIs Node, usar `nodejs_compat`.

---

## 19. Desarrollo Local

Patrón esperado:

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

## 20. Producción

En producción todo vive bajo el mismo origen:

```text
https://dominio.com/
https://dominio.com/api/v1/*
```

No agregar CORS innecesario salvo que exista un consumidor externo real.

---

## 21. Convenciones de Nombres

- Módulos en minúscula: `identity`, `hymns`, `favorites`.
- Schemas Drizzle: `*.schema.ts`.
- Repositorios infraestructura: `*.repository.ts`.
- Hooks React: `use-*.ts` o `use-*.tsx`.
- Clientes externos: `*-client.ts`.
- Middlewares: `require-auth.ts`, `require-role.ts`.

---

## 22. Código Generado

Código generado por herramientas externas debe tratarse como propiedad de la herramienta.

Ejemplos:

- `auth.schema.ts` generado por Better Auth.
- Archivos generados por TanStack Router.
- Tipos generados por Wrangler.

No modificar manualmente salvo instrucción explícita.

---

## 23. Antes de Crear Código Nuevo

Antes de agregar una funcionalidad:

1. Identificar el módulo correcto.
2. Determinar si es dominio, backend o frontend.
3. Revisar si ya existe una abstracción compartida.
4. Evitar duplicar infraestructura.
5. Mantener el cambio pequeño y cohesivo.

---

## 24. Antes de Finalizar una Tarea

Verificar:

- El código compila.
- No se rompieron reglas de dependencia.
- No se modificó código generado innecesariamente.
- Biome no reporta errores.
- Los tests relevantes pasan.
- La estructura respeta `ARCHITECTURE.md`.

---

## 25. Regla de Oro

Si una tarea requiere romper una regla de arquitectura, no lo hagas silenciosamente.

Primero documenta la razón y solicita confirmación.
