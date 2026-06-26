# Especificación Técnica de Arquitectura Base - Monolito Híbrido Modular

Este documento contiene las definiciones de arquitectura, stack tecnológico, decisiones de diseño y el plan de implementación base para el desarrollo de aplicaciones web de alta concurrencia desplegadas en el Edge.

## 1. Stack Tecnológico Base

### Plataforma de Despliegue
* **Servicio:** Cloudflare Workers (con Assets/Static Fetching habilitado).
* **Modelo de Ejecución:** Edge Computing (baja latencia global, escalado instantáneo).

### Herramientas de Desarrollo, Calidad y Pruebas (Tooling)
* **Gestor de Paquetes:** pnpm (instalación ultra rápida mediante enlaces simbólicos, eficiente en disco y óptimo para arquitecturas modulares).
* **Bundler/Dev Server:** Vite (compilación de la SPA, TypeScript y entorno unificado con HMR).
* **Linter & Formatter:** Biome (herramienta unificada ultra rápida para el formateo, linting y análisis estático del código TypeScript/React).
* **Automatización Local:** Husky (gestión de Git Hooks para automatizar la validación de código antes de cada commit).
* **Suite de Testing:** Vitest (entorno de pruebas unificado de velocidad ultra alta para la ejecución de test de dominio, backend y frontend).
* **Entorno Cloudflare:** Wrangler CLI (empaquetado del Worker y despliegue local/remoto).

### Framework de Aplicación (Backend API)
* **Tecnología:** Hono.js
* **Rol:** API REST y Servidor de Archivos Estáticos para la SPA.
* **Prefijo de API:** `/api/v1/*`
* **Lenguaje:** TypeScript puro.

### Autenticación y Seguridad
* **Solución:** Better-auth
* **Características:** Autenticación agnóstica al framework, soporte nativo para TypeScript, plugins de sesión basados en cookies seguras e integración directa con Drizzle ORM.
* **Control de Acceso:** Manejo de roles (RBAC) interconstruido para diferenciar accesos en la plataforma.

### Cliente (Frontend SPA)
* **Librería Core:** React
* **Enrutador:** TanStack Router (enrutamiento seguro basado en tipos en el lado del cliente).
* **Estilos:** Tailwind CSS (Diseño utilitario y responsivo).
* **Componentes de UI:** Shadcn UI (Componentes accesibles basados en Radix UI y estilizados con Tailwind, copiados directamente al repositorio).
* **Gestión de Estado y Servidor:** TanStack Query (React Query) (Manejo asíncrono de peticiones HTTP, caché y mutaciones con la API de Hono).
* **Manejo de Datos Complejos:** TanStack Table (Lógica headless para ordenamiento, filtrado y paginación de datos, integrada visualmente con Shadcn UI).
* **Formularios y Validaciones:** React Hook Form + Zod (Estructura de formularios ligera, eficiente en re-renders y validación mediante esquemas de datos).

### Base de Datos y Persistencia
* **Motor:** Neon PostgreSQL (Serverless Postgres).
* **Driver de Conexión:** `@neondatabase/serverless` (WebSockets optimizados para entornos Edge).
* **ORM:** Drizzle ORM (Mapeo de tipos y control total de consultas SQL nativas).

---

## 2. Decisiones de Arquitectura

### Monolito con SPA Embebida
El proyecto se compila en un único artefacto donde el Worker de Cloudflare intercepta todas las peticiones:
1. Las rutas que coinciden con `/api/v1/*` ejecutan los controladores y endpoints de Hono.
2. Cualquier otra ruta (`/*`) sirve el `index.html` generado previamente por Vite en la carpeta de distribución (`dist`), permitiendo que TanStack Router tome el control del enrutamiento en el navegador (SPA).

### Enfoque de Arquitectura: DDD Híbrido (Monolito Modular)
Se adopta **Domain-Driven Design (DDD)** organizado en **módulos verticales (Bounded Contexts)**. Cada módulo funcional del sistema agrupa la totalidad de su contexto de negocio, estructurado internamente en tres capas con responsabilidades independientes de acuerdo a **Clean Architecture**:

1. **Domain:** Capa pura de TypeScript. Contiene entidades, reglas de negocio esenciales, *Value Objects* e interfaces de repositorios (puertos). No tiene dependencias de frameworks ni herramientas externas.
2. **Backend (Infraestructura):** Controladores de Hono.js, esquemas de tablas de Drizzle ORM e implementaciones de persistencia hacia Neon Postgres (adaptadores/repositorios).
3. **Frontend (Infraestructura):** Componentes visuales de React, estado de cliente, hooks de llamadas a la API (vía TanStack Query) y definición de rutas locales para TanStack Router.

### Reglas de Dependencia de Datos y Aislamiento
* **Aislamiento del Dominio:** La capa de dominio es agnóstica a la infraestructura. Ninguna entidad puede importar código de React, Hono o Drizzle.
* **Desacoplamiento Frontend/Backend:** La capa de frontend bajo ninguna circunstancia realiza consultas directas a la base de datos. Toda interacción se realiza consumiendo los endpoints HTTP expuestos por Hono.
* **Type Safety de Extremo a Extremo:** Se aprovecha el entorno monolítico para compartir los tipos de TypeScript de los controladores del backend hacia los componentes del frontend, garantizando contratos de API validados en tiempo de compilación.

### Calidad de Código Local (Git Hooks con Husky)
Para evitar subir código roto al repositorio, se configura **Husky** mediante un hook de `pre-commit`. Cada vez que el desarrollador intente realizar un commit local, Husky ejecutará automáticamente `pnpm biome check --write`. Si el linter detecta errores críticos o el formateo falla, el commit se abortará inmediatamente.

### Pipeline de CI/CD (GitHub Actions)
El ciclo de vida del despliegue se automatiza mediante GitHub Actions estructurado en etapas secuenciales optimizadas mediante el sistema de almacenamiento en caché nativo de `pnpm`:
1. **Fase de Validación (CI):** Instala librerías vía `pnpm install --frozen-lockfile` y ejecuta en paralelo la verificación estática estricta con Biome (`pnpm biome ci .`) y la suite completa de pruebas automatizadas con Vitest (`pnpm vitest run`).
2. **Fase de Persistencia (CD):** Si las validaciones previas son exitosas, ejecuta de forma aislada las migraciones pendientes en la base de datos de Neon Postgres utilizando `pnpm drizzle-kit migrate`.
3. **Fase de Despliegue (CD):** Compila los recursos estáticos del frontend mediante Vite y despliega el Worker híbrido en la red global de Cloudflare a través de `cloudflare/wrangler-action`.

### Estrategia de Testing Unificado
Se adopta **Vitest** como la herramienta única de testing para todo el repositorio, aprovechando la infraestructura compartida del bundler de Vite:
* **Test de Dominio (TypeScript Puro):** Pruebas unitarias directas sobre las clases de negocio, entidades y value objects.
* **Test de Backend (Hono.js):** Pruebas de endpoints utilizando el método `.request()` nativo de Hono para simular peticiones sin abrir puertos de red.
* **Test de Frontend (React):** Pruebas de componentes lógicos y vistas utilizando `@testing-library/react` combinadas con `happy-dom` o `jsdom`.

### Inyección de Entorno y Variables Tipadas (Cloudflare Bindings)
Las variables de entorno en producción se inyectan a través del contexto de Hono (`c.env`). El objeto de entorno se valida rigurosamente usando esquemas de Zod al arrancar la aplicación para evitar fallos catastróficos en tiempo de ejecución por variables faltantes.

### Control Global de Errores
Se implementa un middleware unificado en Hono a través de `app.onError()`. Cualquier excepción lanzada es capturada, sanitizada para evitar fugas de información sensible de la base de datos, y devuelta en un formato estructurado JSON compatible con los manejadores de errores de TanStack Query en el cliente.

### Control de Concurrencia Crítica
Para operaciones sensibles del negocio con alta probabilidad de colisiones de datos, se utiliza **Bloqueo Pesimista (Pessimistic Locking)** mediante instrucciones `FOR UPDATE` en transacciones nativas de PostgreSQL ejecutadas a través de Drizzle ORM.

### Calidad de Código (Linting y Formateo)
Se unifica el análisis estático bajo Biome. Al compilar en tiempos de milisegundos eliminamos la sobrecarga de herramientas separadas (ESLint/Prettier), asegurando consistencia tanto en los componentes de React como en las rutas de Hono.

### Estrategia de Autenticación (Better-auth)
La autenticación se centraliza en el backend (Hono) montando el manejador de Better-auth en una ruta dedicada (ej. `/api/v1/auth/*`).
* **Seguridad:** Las sesiones se manejan mediante cookies `HttpOnly`, `Secure` y `SameSite=Lax`.
* **Protección de Vistas:** Se implementan middlewares de Hono para proteger endpoints críticos del backend y guardas de rutas (*route guards*) en TanStack Router para restringir vistas en el cliente según el rol del usuario obtenido de la sesión.

---

## 3. Plan de Trabajo e Implementación Base

Este plan incremental mitiga los riesgos técnicos más altos al principio (Fases 1 y 2) antes de desarrollar los módulos del negocio específicos.

### Fase 1: El Esqueleto, Entorno Local y Testing
**Objetivo:** Dejar operativo el "cableado" técnico, entorno de desarrollo, git hooks y el pipeline automatizado.
- [ ] **Paso 1.1:** Inicialización del repositorio con **pnpm**, configuración de **Biome** (linter/formatter), setup de **Tailwind CSS** junto con la CLI de **Shadcn UI** e inicialización del entorno de pruebas unitarias con **Vitest**.
- [ ] **Paso 1.2:** Instalación y configuración de **Husky** para interceptar el hook de `pre-commit` y forzar la validación automatizada de Biome localmente antes de guardar cambios.
- [ ] **Paso 1.3:** Configuración de `vite.config.ts` y Wrangler para levantar el frontend (React/TanStack Router/TanStack Query) y el backend (Hono) en un único comando con redirección de rutas y API (`/api/v1/*`). Implementación del tipado del contexto de Hono (`c.env`) y el middleware de manejo global de errores. Escribir un test de humo (*smoke test*) con Vitest que verifique el levantamiento correcto de la API.
- [ ] **Paso 1.4:** Configuración del cliente de Drizzle y el driver `@neondatabase/serverless` para conectar el Worker con Neon PostgreSQL. Configuración de los scripts de migraciones locales (`drizzle-kit`) y ejecución de una migración de prueba.
- [ ] **Paso 1.5:** Configuración del archivo de workflow de **GitHub Actions** (`.github/workflows/deploy.yml`) estructurado en bloques secuenciales utilizando las acciones de caché de pnpm: Instalación, validación con Biome/Vitest, migración remota con Drizzle Kit y despliegue del Worker en Cloudflare.

### Fase 2: Cimientos de Identidad (Autenticación y RBAC)
**Objetivo:** Implementar la infraestructura de seguridad y control de acceso base antes de la lógica de negocio.
- [ ] **Paso 2.1:** Configuración de **Better-auth** en el backend (Hono), generación y ejecución de las migraciones de las tablas de usuarios y sesiones en Postgres vía Drizzle. Escribir test unitarios para asegurar el bloqueo de rutas mediante el middleware de sesión de Better-auth.
- [ ] **Paso 2.2:** Configuración de los Roles base requeridos por la plataforma utilizando el sistema RBAC de Better-auth.
- [ ] **Paso 2.3:** Integración con el Frontend: Pantalla de Login base en React y configuración de guardas de ruta (*route guards*) en TanStack Router para restringir accesos según el estado de la sesión. Testear componentes de login simulando el DOM.

### Fase 3: Implementación de Módulos Verticales de Negocio
**Objetivo:** Desarrollar las funcionalidades del negocio aplicando la separación DDD + Clean Architecture diseñada.
- [ ] **Paso 3.1 (Dominio):** Modelado del Dominio (`modules/<contexto>/domain`): Creación de entidades puras de TypeScript, reglas de negocio complejas y contratos de repositorios (interfaces). **Escribir cobertura estricta de test de unidad con Vitest.**
- [ ] **Paso 3.2 (Backend):** Infraestructura Backend (`modules/<contexto>/backend`): Creación de endpoints en Hono, esquemas relacionales de Drizzle e implementación de contratos de persistencia. **Escribir test de endpoints simulando peticiones HTTP con `.request()`.**
- [ ] **Paso 3.3 (Frontend):** Infraestructura Frontend (`modules/<contexto>/frontend`): Creación de vistas en React utilizando componentes de Shadcn UI, manejo de estado asíncrono con TanStack Query/Table y formularios reactivos validados con Zod. **Escribir test funcionales de UI (testing-library) verificando el comportamiento de las vistas ante cargas y errores.**
