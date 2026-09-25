# Plan de implementación — Operador de Carpeta Ciudadana

> **Versión:** 1.0 — **Estado:** PROPUESTO, pendiente de aprobación del equipo.
> **Fecha:** 2026-09-25
> **Base:** `docs/analysis/02-arquitectura-propuesta.md` (v2.0) + precisiones posteriores aprobadas por el equipo (§3.2).
> **Regla:** este plan **no introduce decisiones arquitectónicas**. Las propuestas de *implementación* que requieren aprobación se marcan **[APROBAR H-xx]** y se consolidan en §13.

---

## 1. Objetivo

Construir un operador de Carpeta Ciudadana **distribuido, funcional y demostrable** que:

1. cumpla los cuatro requisitos de `05`: registro, login, carga de documentos y autenticación de documentos vía GovCarpeta;
2. interopere con otros operadores mediante TransferCitizen (entrante y saliente) según `04`;
3. demuestre **aislamiento de fallas** entre sus tres microservicios usando Docker Compose;
4. lo haga con la infraestructura mínima aprobada, sin plataforma empresarial.

## 2. Alcance

### 2.1 Clasificación

| Clase | Elementos |
|---|---|
| **Obligatorio para que funcione** | CU registro (validateCitizen + registerCitizen + usuario de Keycloak + correo institucional); login OIDC + PKCE; carga, lista, detalle y descarga de documentos con SHA-256; autenticación con `authenticateDocument` mediante URL de capacidad; `/api/descargas/{token}` con token aleatorio, TTL e invalidación; TransferCitizen entrante (recepción, validación, importación, registro, `confirmAPI`); TransferCitizen saliente (preparar, congelar, desafiliar, enviar, esperar, purgar solo con confirmación válida); máquinas de estado persistentes con reintentos e idempotencia básica; nunca eliminar por timeout; `GET /api/operadores` (directorio en caché); validación JWT independiente con el claim `citizen_id`; NGINX con allowlist de rutas; una BD por servicio; MinIO privado; health checks `live` / `ready`; validación básica de URLs externas; timeouts; límites de tamaño; validación de payloads; Docker Compose; hostname público estable |
| **Importante si es sencillo** | `X-Request-Id` extremo a extremo y logs JSON con `citizen_id` enmascarado; cuerpo JSON uniforme para 502/503/504 en NGINX; rate limit en NGINX para rutas externas y registro; historial de intentos de autenticación; `HEAD` en `/api/descargas`; reconciliación programada de ciudadanos `PENDIENTE`; grupo de health informativo de dependencias remotas; seguir redirecciones (máx. 3) revalidando cada salto; token de servicio en `/internal/*` **[APROBAR H-04]** |
| **Solo documentación** | Defensas SSRF avanzadas (fijar la IP tras resolver, DNS rebinding, allowlist de hosts por directorio); "confirmación inferida" desde GovCarpeta; extensiones opcionales del contrato (`transferId`, `documents`, `citizenAddress`, `message`); alta disponibilidad (réplicas de NGINX y Keycloak); manifiestos de Kubernetes; Vault; pila de observabilidad; MFA |
| **Fuera de alcance** | Kubernetes, Kong, RabbitMQ, Kafka, Vault, service mesh, microservicio gateway, microservicios adicionales, BD o almacenamiento compartidos, MFA, observabilidad empresarial, mTLS/PKI/OAuth entre operadores; funciones de `02` no pedidas: solicitudes y paquetes, premium, analítica, notificaciones, carpeta institucional, cuota de no certificados, verificación criptográfica de firma, carga automática de la cédula (RF-07); pantallas administrativas |

### 2.2 Qué NO se implementará (resumen explícito)

- Ningún componente de la lista "fuera de alcance".
- Ningún envío de extensiones del contrato TransferCitizen: se aceptan al recibir, pero no se envían.
- Ninguna purga automática por timeout, ni "confirmación inferida".
- Ninguna UI para `registerOperator`, `registerTransferEndPoint`, limpieza de pruebas ni revisión de traslados en `REQUIERE_REVISION`: son procedimientos operativos con script y runbook.
- Ninguna validación de negocio **solo** en el frontend.

---

## 3. Estado actual del repositorio y precisiones vigentes

### 3.1 Inventario (2026-09-25)

| Elemento | Estado |
|---|---|
| `resources/01` a `05` | Presentes, solo lectura, sin cambios desde la auditoría |
| `docs/analysis/02-arquitectura-propuesta.md` | v2.0 completa. Encabezado aún en "PROPUESTA PARA APROBACIÓN" (ver I-02) |
| `docs/analysis/technical-spikes/` | Vacío: **ningún spike ejecutado** |
| `docs/decisions/ADR-001` a `ADR-005` | **Archivos vacíos (0 bytes)**. Sin contenido que revisar ni que contradiga algo |
| `docs/implementation/api-contracts.md` | **Vacío**: no hay contratos detallados |
| `docs/implementation/implementation-plan.md` | Este documento |
| `docker-compose.yml`, `.env.example`, `.gitignore`, `src/nginx/nginx.conf` | **Vacíos** (andamiaje) |
| `CLAUDE.md`, `README.md` | Vacíos |
| `src/frontend`, `src/ms-ciudadano`, `src/ms-documentos`, `src/ms-interoperabilidad` | Carpetas vacías |
| `tests/integration`, `tests/ms-*` | Carpetas vacías |
| Control de versiones | **El directorio no es un repositorio git** |
| Herramientas en la máquina revisada | Docker 29.3 / Compose v5.1 instalados, **pero el daemon de Docker Desktop no estaba corriendo**; Java 21.0.12 ✔; **Maven no instalado**; **Node v14.21 (demasiado antiguo para las herramientas actuales de React/Vite)**; git 2.53 ✔; sin `cloudflared` ni `ngrok` |

### 3.2 Precisiones posteriores aprobadas (prevalecen sobre `02`)

| Tema | `02-arquitectura-propuesta.md` | Decisión vigente para implementar |
|---|---|---|
| Defensa SSRF | §16.2 incluye fijar la IP tras resolver, DNS rebinding y allowlist por directorio | **Validación básica:** esquema http/https; sin credenciales en la URL; puertos 80/443 (lista configurable); rechazar hosts que resuelvan a loopback, privadas, link-local (incluida 169.254.169.254) o 0.0.0.0; timeouts; límite de bytes; redirecciones limitadas. Lo avanzado queda como **documentación** |
| Confirmación perdida (DA-18) | Pendiente de preguntar al profesor si se permite "confirmación inferida" | **Resuelto:** nunca se purga sin confirmación válida → `VERIFICANDO` / `REQUIERE_REVISION` |
| Hostname público | DE-02 pendiente | **Aprobado:** hostname o túnel **estable**. El mecanismo concreto sigue pendiente **[APROBAR H-01]** |
| Seguridad entre operadores | §10.8 | **Controles prácticos:** token en `confirmAPI` + validación contra el estado + validaciones de payload. Nada de mTLS, OAuth ni PKI |
| Pantallas | §20 | Login, registro, inicio, documentos, carga, detalle/descarga, traslado, estado del traslado |

---

## 4. Inconsistencias y hallazgos

| ID | Hallazgo | Impacto | Tratamiento propuesto |
|---|---|---|---|
| I-01 | Los ADR-001 a 005, `api-contracts.md`, `docker-compose.yml`, `.env.example`, `.gitignore` y `nginx.conf` existen pero están **vacíos** | No hay decisiones registradas ni contratos detallados; no bloquea el inicio | Redactar contratos en F0. Redactar ADRs cuando se aprueben (§12) |
| I-02 | `02-arquitectura-propuesta.md` dice "PROPUESTA"; el equipo la declara aprobada con precisiones (§3.2) que el archivo no refleja | Un lector futuro podría implementar la SSRF avanzada o dudar de DA-18 | **[APROBAR H-13]** Actualizar el estado del documento y anotar las precisiones |
| I-03 | `tests/ms-*` separados de `src/ms-*` choca con la convención de Maven y Quarkus (pruebas en `src/test/java` de cada proyecto) | Configuración forzada si se insiste en separar | **[APROBAR H-06]** Pruebas unitarias y de componente dentro de cada servicio; `tests/` solo para pruebas de caja negra contra el stack |
| I-04 | DA-21 (token de servicio en `/internal/*`) está en `02`, pero el alcance práctico de seguridad del equipo no lo menciona | Decidir si se implementa | **[APROBAR H-04]** |
| I-05 | DA-22 (activación por correo con Mailpit) introduce un contenedor no listado en la arquitectura aprobada | Nueva tecnología sin aprobación | **[APROBAR H-03]** |
| I-06 | "Si GovCarpeta falla, las operaciones deben quedar pendientes o reintentarse" frente a `02` §13.3, donde el registro **falla cerrado** (503, sin persistir) si no se puede verificar la afiliación | Aparente conflicto | **Interpretación [APROBAR H-12]:** el registro no puede quedar "pendiente" sin verificar la afiliación única (CASO-01). Se reintenta a petición del usuario; solo queda `PENDIENTE` si el fallo ocurre en `registerCitizen`. La autenticación de documentos queda en `ERROR_AUTENTICACION`, reintentable manualmente. Los traslados sí se reintentan automáticamente |
| I-07 | El proyecto no está bajo control de versiones | Impide trabajar en paralelo y revisar cambios | **[APROBAR H-08]** Crear el repositorio git antes de F1 |
| I-08 | Herramientas: sin Maven, Node 14, daemon de Docker apagado, sin cliente de túnel | Bloquea compilar y ejecutar localmente si no se resuelve | **[APROBAR H-09]** Compilar todo en Docker (multi-stage) y usar Maven Wrapper; túnel como contenedor; iniciar Docker Desktop |
| I-09 | El stack (Java 21 + Quarkus 3, React + TS) está en `02` (DA heredadas de AD-04), pero el prompt del equipo no lo repite | Ninguno si se mantiene | Se asume `02`. Confirmar en §13 |
| I-10 | Las preguntas al profesor de `02` §22.3 siguen sin respuesta | Q5 (`registerOperator`) y Q3 (disponibilidad pública) afectan fases concretas; Q1 quedó resuelta para DA-18; Q2 la asumió el equipo | Ver bloqueadores §5 y riesgos §11 |
| I-11 | `04` sigue con autoría ambigua (profesor o compañero) | Ninguno para implementar: se trata como contrato entre equipos | Solo registro |
| I-12 | Para probar TransferCitizen de extremo a extremo hacen falta **dos operadores** y un GovCarpeta con estado de afiliación coherente | La arquitectura no define la estrategia de prueba | **[APROBAR H-05]** |
| I-13 | Probar APIs con JWT de forma automatizada requiere obtener tokens sin navegador; el cliente PKCE no lo permite | Complica las pruebas automáticas | **[APROBAR H-07]** Cliente de pruebas solo en el realm de desarrollo |

---

## 5. Bloqueadores reales

**Para iniciar F0 (contratos y preparación) no hay bloqueadores.** Los bloqueadores son por fase:

| ID | Bloqueador | Bloquea | Cómo se resuelve |
|---|---|---|---|
| BL-01 | Daemon de Docker no activo | F1 en adelante | Iniciar Docker Desktop (acción del equipo) |
| BL-02 | Sin repositorio git | Trabajo en paralelo (a partir de F1) | H-08 |
| BL-03 | `api-contracts.md` vacío | Trabajo en paralelo de servicios y frontend; toda integración | Entregable de F0 |
| BL-04 | Mecanismo de hostname estable sin decidir (proveedor, dominio, cuenta) | F2 (spikes externos), pruebas reales con GovCarpeta de F6 y F11 | H-01 |
| BL-05 | Datos del operador y quién ejecuta `registerOperator` (**irreversible** en un directorio compartido) | Pruebas reales de registro (F4b) y de traslado (F11) | H-02 + SP-07 |
| BL-06 | Activación de ciudadanos recibidos por traslado | Cierre de F9 | H-03 |
| BL-07 | Estrategia para probar traslados entre dos operadores | F11 | H-05 |

El desarrollo contra **GovCarpeta simulado (WireMock)** permite avanzar en F4 a F10 sin que BL-04 y BL-05 estén resueltos.

---

## 6. Fases

### 6.0 Resumen y orden

| Fase | Nombre | Depende de | ¿En paralelo con? |
|---|---|---|---|
| F0 | Preparación: repositorio, contratos, configuración, decisiones | — | — |
| F1 | Plataforma base: Compose, redes, PostgreSQL ×3, MinIO, Keycloak, NGINX, esqueletos | F0 | F2 (parcial) |
| F2 | Frontera pública estable + spikes externos | F1, H-01 | F3, F4, F5 |
| F3 | Frontend base + login OIDC | F1 | F4, F5 |
| F4 | ms-ciudadano: registro y perfil (+ GovCarpeta de afiliación) | F0, F1 | F3, F5 |
| F5 | ms-documentos: carga, lista, detalle, descarga | F0, F1 | F3, F4 |
| F6 | URLs de capacidad + autenticación de documentos con GovCarpeta | F5 (+ F2 para la prueba real) | F7 (parte de ciudadano) |
| F7 | APIs `/internal/*` de ms-ciudadano y ms-documentos para traslados | F4, F5 | F8 |
| F8 | ms-interoperabilidad base: BD, motor de estados, directorio, cliente HTTP con validación básica | F0, F1 | F7 |
| F9 | TransferCitizen entrante | F7, F8 | F10 |
| F10 | TransferCitizen saliente + TransferCitizenConfirm | F7, F8 | F9 |
| F11 | Integración E2E: dos instancias propias y luego otro equipo | F9, F10, F2, H-05 | — |
| F12 | Pruebas de fallas, aislamiento y recuperación | F11 (casi todo es posible desde F6) | — |
| F13 | Demostración, runbooks y documentación final | F12 | — |

### 6.0.1 Cambios respecto al orden de referencia

1. **La frontera pública y los spikes pasan a F2, al inicio.** `02` §23 los exige antes de fijar contratos. Si SP-01 falla (GovCarpeta no acepta nuestra URL), cambia el mecanismo de acceso a documentos. Descubrirlo en la fase 9 obligaría a rehacer trabajo.
2. **El frontend se divide:** una base en F3 y luego cada pantalla dentro de su slice (F4, F5, F6, F10), en lugar de una fase final única. Así cada slice es demostrable y el riesgo de Keycloak detrás del proxy (SP-08) se valida temprano. Es lo que significa "implementación vertical".
3. **La integración de GovCarpeta de ciudadano (paso 8 de referencia) va dentro de F4.** El registro no está completo sin `validateCitizen` y `registerCitizen`.
4. **Nueva fase F7 (APIs internas para traslados) antes de interoperabilidad.** `ms-interoperabilidad` depende de ellas. Construirlas dentro de sus servicios dueños preserva la propiedad de los datos.
5. **Los pasos 12 a 14 de referencia se agrupan por dirección (F9 entrante, F10 saliente + confirmación).** Estados, reintentos e idempotencia **son** la lógica de cada flujo, no una capa que se agrega después. El motor común (reclamo de trabajos, retroceso) se construye en F8.

No se agregan fases de arquitectura nuevas.

---

### F0 — Preparación

| Aspecto | Detalle |
|---|---|
| **Objetivo** | Dejar listo todo lo que permite trabajar en paralelo sin contradicciones |
| **Tareas** | 1) Crear el repositorio git, la estrategia de ramas y el `.gitignore` (H-08). 2) Redactar **`api-contracts.md`** (§7). 3) Catálogo de variables de entorno (§9) y `.env.example` (sin secretos). 4) Resolver las decisiones H-01 a H-13. 5) Definir el rango de ids de prueba (H-10). 6) SP-07 en su parte **no destructiva** (semántica de `validateCitizen` con ids de prueba). 7) Preparar los datos para `registerOperator` (sin ejecutarlo hasta aprobar H-02). 8) Actualizar el estado de `02` (H-13). 9) Proponer los ADRs (§12) |
| **Entregables** | Repositorio versionado; `api-contracts.md` v1; `.env.example`; decisiones registradas; notas de SP-07 en `docs/analysis/technical-spikes/` |
| **Criterio de aceptación** | Todos los endpoints de `02` §7 y §8 tienen request, response, códigos y errores definidos; formato de error común; nombres de claims, headers y variables fijados; decisiones H bloqueantes resueltas |
| **Pruebas** | Revisión cruzada de contratos por quien implementará cada lado (consumidor y proveedor) |

### F1 — Plataforma base

| Aspecto | Detalle |
|---|---|
| **Objetivo** | Levantar con un comando toda la infraestructura aprobada y los tres servicios vacíos pero saludables |
| **Componentes** | `docker-compose.yml`. **Redes:** `edge`, `servicios`, `net-ciudadano`, `net-documentos`, `net-interop`, `net-minio`. **Contenedores:** `pg-ciudadano`, `pg-documentos`, `pg-interop`, MinIO (+ creación del bucket privado), Keycloak (realm `carpeta` importado desde JSON versionado: cliente `carpeta-web` PKCE, clientes confidenciales `ms-ciudadano` y `ms-interoperabilidad`, rol `ciudadano`, mapper `citizen_id`, audiencia `carpeta-api`, vigencias, fuerza bruta, ruta relativa `/auth`), NGINX (allowlist de rutas, resolver 127.0.0.11 con `proxy_pass` por variable, `X-Request-Id`, límites de tamaño, 404 para lo no publicado, bloqueo de `/internal`, `/q`, `/auth/admin` y del realm `master`). **Esqueletos Quarkus** de los tres servicios, solo con `/q/health/live` y `/q/health/ready` (readiness = su BD, y MinIO en documentos). Esqueleto de la SPA (página estática) |
| **Reglas** | Solo NGINX publica un puerto. `depends_on` de cada servicio **solo** hacia su BD (y MinIO). Imágenes con versión fijada. Compilación en Docker multi-stage (H-09) |
| **Criterio de aceptación** | `docker compose up` deja todo `healthy`; `docker compose stop ms-ciudadano` no afecta la salud de los demás ni el arranque de NGINX; desde el host solo responde el puerto de NGINX; `/internal/x`, `/q/health` y `/auth/admin` devuelven 404 vía NGINX; login de un usuario creado a mano en Keycloak funciona vía `/auth` |
| **Pruebas** | Script de verificación de F1 (health, puertos expuestos, rutas bloqueadas, parada de un servicio) |

### F2 — Frontera pública estable + spikes externos

| Aspecto | Detalle |
|---|---|
| **Objetivo** | Confirmar que Internet nos alcanza por un hostname estable y que las premisas de la arquitectura se cumplen |
| **Componentes** | Túnel como contenedor hacia NGINX (H-01); `PUBLIC_BASE_URL`; endpoints **desechables** de spike en un servicio de prueba (no forman parte del producto y se eliminan) |
| **Spikes** | SP-02 y SP-03 (`transferCitizen` y `transferCitizenConfirm` alcanzables desde fuera; el query `?t=` llega intacto), SP-04 (nada interno expuesto), SP-08 (issuer de Keycloak detrás del proxy y del túnel), SP-09 (estabilidad del host), SP-01 (GovCarpeta llama a `authenticateDocument` con una URL del tipo `/api/descargas/{token}` que sirve un archivo de prueba; ¿descarga?), SP-06 (con al menos un equipo: ¿respetan `confirmAPI` literal? ¿códigos? ¿formato de `urlDocuments`?) |
| **Entregables** | Resultados en `docs/analysis/technical-spikes/SP-xx.md` con evidencia (logs, capturas) |
| **Criterio de aceptación** | Todos los spikes con resultado documentado. **Si SP-01 falla, se detiene y se escala** la decisión de acceso a documentos (§14 de `02`: alternativas S3 o VM) **antes** de F6 |
| **Pruebas** | Las de cada spike (`02` §23) |

### F3 — Frontend base + login

| Aspecto | Detalle |
|---|---|
| **Objetivo** | SPA navegable con login y logout reales |
| **Componentes** | React + TS (compilado en Docker, servido por NGINX); keycloak-js con PKCE; tokens en memoria; guardas de ruta; layout; componente de aviso por sección ante 502/503/504; cliente HTTP con `Authorization` y `X-Request-Id` |
| **Criterio de aceptación** | Login → pantalla de inicio → logout, vía NGINX (y vía túnel si F2 terminó); refresco del token; sin CORS (mismo origen) |
| **Pruebas** | Compilación y lint; lista de verificación manual (login, token vencido, logout) |

### F4 — ms-ciudadano: registro y perfil

| Aspecto | Detalle |
|---|---|
| **Objetivo** | CU-01 completo y `GET /api/ciudadanos/me` |
| **Componentes** | `ciudadano_db` (ciudadano, con estados `PENDIENTE` / `ACTIVO` / `EN_TRASLADO`); adaptador de Registraduría simulada (ids configurables para rechazo); adaptador GovCarpeta de afiliación (`validateCitizen`, `registerCitizen`, `unregisterCitizen`) con traducción a `DISPONIBLE` / `AFILIADO_A_ESTE` / `AFILIADO_A_OTRO`; cliente Admin de Keycloak (crear deshabilitado, habilitar, eliminar); generación del correo institucional (patrón de `02` §7.4 de la v1, conservado en §13.3 de la v2); flujo de `02` §13.3 con reconciliación al reintentar; validación JWT (`citizen_id`, rol `ciudadano`); pantallas de **registro** e **inicio/perfil** |
| **Sub-fases** | **F4a** contra WireMock (sin dependencia externa). **F4b** prueba de humo con GovCarpeta real (requiere H-02 ejecutado y H-10) y limpieza posterior |
| **Criterio de aceptación** | Id nuevo → 201, `ACTIVO`, correo generado, login posible. Id afiliado a otro → 409 sin residuos. Id repetido → 409. Registraduría rechaza → 422. Keycloak caído → 503 sin fila. `registerCitizen` con timeout → `PENDIENTE`; el reintento reconcilia. No existe forma de modificar el correo. La contraseña no aparece en logs ni en la BD |
| **Pruebas** | Unitarias (generación de correo, traducción de códigos); de componente con BD real (Dev Services) + WireMock (200/204/201/501/500/timeout) + Keycloak de pruebas; F4b manual documentada |
| **Importante si es sencillo** | Reconciliación programada de `PENDIENTE` |

### F5 — ms-documentos: carga, lista, detalle y descarga

| Aspecto | Detalle |
|---|---|
| **Objetivo** | CU-03 y la consulta y descarga de documentos propios |
| **Componentes** | `documento_db` (documento, carpeta); cliente MinIO (credenciales exclusivas); carga multipart con validación de título, tamaño (20 MB, también en NGINX) y tipo por magic bytes (PDF, JPEG, PNG); SHA-256 en streaming; clave `docs/{uuid}`; propiedad por `citizen_id` (404 para documentos ajenos); descarga con `attachment` + `nosniff`; `409 CARPETA_CONGELADA` si la carpeta está congelada; pantallas **documentos**, **carga** y **detalle/descarga** |
| **Criterio de aceptación** | Carga válida → 201 y aparece en la lista; el hash de la descarga coincide. Sin título, más de 20 MB o `.exe` renombrado → 400 / 413 / 415. A no ve ni descarga documentos de B (404). MinIO caído → 503 sin metadatos huérfanos. Todo funciona con `ms-ciudadano` detenido |
| **Pruebas** | Unitarias (detección de tipo, hash); de componente (BD + MinIO en contenedores); seguridad (acceso ajeno, sin token, token alterado) |

### F6 — URLs de capacidad + autenticación de documentos

| Aspecto | Detalle |
|---|---|
| **Objetivo** | CU-04 completo |
| **Componentes** | Tabla `enlace_temporal` (hash del token, propósito, expiración, revocado, accesos); `GET` y `HEAD /api/descargas/{token}` (404 si es inválido, vencido o revocado; `no-store`); token aleatorio de 256 bits sin datos personales; TTL de 15 min para autenticación; adaptador GovCarpeta `authenticateDocument`; tabla `intento_autenticacion`; estados `CARGADO → EN_AUTENTICACION → AUTENTICADO / NO_AUTENTICADO / ERROR_AUTENTICACION` con transición condicional (409 ante doble clic) y recuperación de `EN_AUTENTICACION` de más de 2 min; 200 → `AUTENTICADO`, 204 → `NO_AUTENTICADO`, 501 → error (no es veredicto), 500 o timeout → reintentable; atender descargas en paralelo mientras se espera a GovCarpeta; botón **Autenticar/Reintentar** e historial en el detalle |
| **Depende de** | F5; F2/SP-01 para la prueba real |
| **Criterio de aceptación** | Con GovCarpeta real: documento `AUTENTICADO` y acceso registrado en `enlace_temporal` (si GovCarpeta descarga). Token vencido o alterado → 404. Funciona con `ms-interoperabilidad` detenido |
| **Pruebas** | WireMock (200/204/501/500/timeout); expiración; revocación; prueba real documentada |

### F7 — APIs internas para traslados (en los servicios dueños)

| Aspecto | Detalle |
|---|---|
| **Objetivo** | Exponer, **sin publicarlas**, las operaciones que `ms-interoperabilidad` necesita, cada una en su dominio |
| **ms-ciudadano** | `GET /internal/afiliaciones/{id}`; `POST /internal/ciudadanos/{id}/traslado-saliente`; `POST …/desafiliacion` (`unregisterCitizen`); `POST …/reafiliacion` (compensación); `DELETE /internal/ciudadanos/{id}` (purga del usuario de Keycloak + fila); `POST /internal/ciudadanos/traslado-entrante` (alta sin contraseña + `registerCitizen`, idempotente por `transferId`, dirección provisional si falta, según SP-07) |
| **ms-documentos** | `PUT`/`DELETE /internal/carpetas/{id}/congelamiento`; `POST …/enlaces-traslado` (URLs de capacidad con TTL de 60 min); `POST …/importaciones` (flujo binario + metadatos, `Idempotency-Key`, tipos libres servidos como `attachment`); `DELETE …/importaciones/{transferId}`; `DELETE /internal/carpetas/{id}` (objetos + metadatos + revocación de enlaces) |
| **Autenticación** | Red `servicios` + (si se aprueba H-04) token de servicio con rol `interop-interno` |
| **Criterio de aceptación** | Todas idempotentes (repetir no cambia el resultado); inaccesibles vía NGINX; con H-04, sin token → 401 |
| **Pruebas** | De componente por endpoint, incluida la repetición y el orden inverso |

### F8 — ms-interoperabilidad base

| Aspecto | Detalle |
|---|---|
| **Objetivo** | Infraestructura interna del servicio sobre la que se construyen F9 y F10 |
| **Componentes** | `interop_db` (traslado con índice único parcial `(citizen_id, dirección)` para traslados no finales, versión optimista; traslado_documento; operador_cache); **motor de trabajos**: proceso programado cada 10 s que reclama traslados con `proximo_intento_en <= ahora` con bloqueo de fila, ejecuta el paso del estado actual, persiste y calcula el retroceso; clientes internos hacia ciudadano y documentos (timeouts 2 s / 25 s; token de servicio si H-04); **cliente HTTP externo** con validación básica de URLs (§3.2), timeouts y límite de bytes; directorio `getOperators` con caché de 5 min y lectura tolerante (`_id`, `operatorName`, `transferAPIURL`); `GET /api/operadores` (filtra los que no tienen URL y a nosotros mismos) |
| **Criterio de aceptación** | El motor reanuda un traslado tras reiniciar el contenedor; dos procesos no ejecutan el mismo paso; el validador rechaza `localhost`, `127.0.0.1`, `10.x`, `192.168.x`, `169.254.169.254`, `file://`, `user@host` y puertos no permitidos; `GET /api/operadores` responde con GovCarpeta simulado |
| **Pruebas** | Unitarias del validador de URLs y del cálculo de retroceso; de componente del motor (reinicio y concurrencia) |

### F9 — TransferCitizen entrante

| Aspecto | Detalle |
|---|---|
| **Objetivo** | Recibir ciudadanos de otros operadores según `04` y `02` §10.4–10.6 |
| **Componentes** | `POST /api/transferCitizen` público: validación de esquema (id numérico o string numérico, `citizenName`, `citizenEmail`, `urlDocuments` como mapa de string o lista de URLs, `confirmAPI`), límites (50 documentos), validación de URLs, idempotencia por ciudadano, **202**; pasos `RECIBIDA → IMPORTANDO → REGISTRANDO → CONFIRMANDO → COMPLETADA` y `REVIRTIENDO → NOTIFICANDO_FALLO → FALLIDA`; espera de 5 min si el origen aún no desafilió; descarga por URL (3 intentos, 20 MB, 60 s) y envío a `ms-documentos`; alta vía `ms-ciudadano`; `confirmAPI` con `req_status` 1/0 y reintentos; reenvío de la confirmación ante un duplicado posterior a `COMPLETADA`; activación del ciudadano recibido según H-03 |
| **Criterio de aceptación** | Payload válido → 202 y, tras el procesamiento, ciudadano `ACTIVO` + documentos importados + confirmación 1 recibida por el origen simulado. Duplicado en curso → 202 sin segundo traslado. Duplicado completado → 200 + reenvío. Ciudadano aún afiliado → confirmación 0 tras la espera. Descarga fallida → reversión + confirmación 0. `confirmAPI` inválida → 400. Con `ms-ciudadano` o `ms-documentos` detenidos, el traslado **espera** y termina cuando vuelven |
| **Pruebas** | De componente con WireMock simulando al operador origen (URLs de documentos, `confirmAPI`) y a GovCarpeta |

### F10 — TransferCitizen saliente + TransferCitizenConfirm

| Aspecto | Detalle |
|---|---|
| **Objetivo** | Trasladar a nuestros ciudadanos a otro operador, según `04` y `02` §10.2–10.3 |
| **Componentes** | `POST /api/traslados` (JWT; titular = `citizen_id`; destino válido del directorio; 409 si hay un traslado en curso; **202**); `GET /api/traslados/actual`; pasos `SOLICITADA → PREPARADA → DESAFILIANDO → ENVIANDO → ESPERANDO_CONFIRMACION → PURGANDO → COMPLETADA`, con `VERIFICANDO` como punto único de decisión y `CANCELANDO/CANCELADA`, `COMPENSANDO/REVERTIDA` y `REQUIERE_REVISION`; `ENVIANDO` persistido **antes** del POST; URLs de capacidad nuevas en cada reintento; `confirmAPI = PUBLIC_BASE_URL/api/transferCitizenConfirm?t=<token>` (se guarda solo el hash); plazo de confirmación configurable (30 min en la demo); `POST /api/transferCitizenConfirm` público con los códigos de `02` §10.10: solo tiene efecto con un traslado activo en un estado que la admita; token válido (o, si el cliente del otro equipo descarta el query, la política de respaldo definida tras SP-06); duplicados → 200 sin efecto; **ninguna transición a `PURGANDO` salvo confirmación 1 válida**; pantallas **Trasladar mi carpeta** y **Estado del traslado**; aviso de carpeta congelada |
| **Criterio de aceptación** | Traslado exitoso contra un destino simulado → purga completa de ambos dominios. Confirmación 0 → `VERIFICANDO` → `REVERTIDA` con el ciudadano `ACTIVO` y la carpeta descongelada. Plazo vencido → `VERIFICANDO` → `REVERTIDA` o `REQUIERE_REVISION`, **nunca** purga. Confirmación falsa (sin traslado activo o con token incorrecto) → 404 sin efecto. Confirmación que llega antes de la respuesta al POST → aceptada. Reinicio en cada estado → reanuda |
| **Pruebas** | De componente con WireMock simulando al destino y a GovCarpeta; una prueba por transición de la tabla de estados |

### F11 — Integración de extremo a extremo

| Aspecto | Detalle |
|---|---|
| **Objetivo** | Demostrar la interoperabilidad real |
| **Etapas** | **11a** Recorrido completo con un solo operador (registro → login → carga → autenticación) vía el hostname público. **11b** Traslado A→B entre **dos instancias propias** (dos proyectos de Compose) según la estrategia H-05. **11c** Traslado en ambas direcciones con **al menos un equipo real**, coordinado |
| **Criterio de aceptación** | 11a con GovCarpeta real. 11b en ambas direcciones, incluido un caso de fallo del destino. 11c documentado (éxito, o incompatibilidades reportadas con evidencia) |
| **Pruebas** | Guiones en `tests/integration/` |

### F12 — Pruebas de fallas, aislamiento y recuperación

| Aspecto | Detalle |
|---|---|
| **Objetivo** | Evidenciar la matriz `02` §17.1 |
| **Escenarios** | `stop` de `ms-ciudadano`, `ms-documentos`, `ms-interoperabilidad`, Keycloak, MinIO, cada `pg-*`, NGINX; GovCarpeta simulado caído o lento; destino caído a mitad de un traslado; muerte de `ms-documentos` durante una autenticación; reinicio de `ms-interoperabilidad` en cada estado |
| **Criterio de aceptación** | Para cada escenario se cumple exactamente la columna "sigue funcionando / deja de funcionar" de `02` §17.1; los demás servicios siguen `healthy`; el `start` recupera sin reiniciar nada más; los traslados pendientes avanzan solos; ningún dato de origen se elimina sin confirmación |
| **Pruebas** | Script por escenario en `tests/integration/fallas/` con un resultado esperado verificable |

### F13 — Demostración y documentación final

| Aspecto | Detalle |
|---|---|
| **Entregables** | `README.md` (cómo levantar el sistema, configuración, arquitectura en una página); runbooks: `registerOperator`, `registerTransferEndPoint` (volver a ejecutarlo si cambia el host), limpieza de ciudadanos de prueba, resolución de `REQUIERE_REVISION`; ADRs aprobados; guion de demo (flujos + apagado de servicios); evidencias de F11 y F12 |
| **Criterio de aceptación** | Un integrante que no participó puede levantar el sistema y ejecutar la demo siguiendo el README |

---

## 7. Contratos necesarios antes de integrar (contenido de `api-contracts.md`)

| Contrato | Necesario antes de | Contenido mínimo |
|---|---|---|
| Convenciones comunes | F4 y F5 | Formato de error JSON (`{codigo, mensaje, requestId}`), `X-Request-Id`, `Idempotency-Key`, formato de fechas, enmascaramiento en logs |
| Claims del JWT | F3, F4, F5 | `citizen_id`, `aud`, roles, vigencias |
| APIs del ciudadano (`02` §8.1) | F3 a F6, F10 (paralelismo frontend/backend) | Request y response con ejemplos, códigos y errores por endpoint |
| APIs `/internal/*` (`02` §8.2) | F7 y F8 | Ídem + garantías de idempotencia + autenticación (H-04) |
| APIs externas | F6, F9, F10 | `transferCitizen` y `transferCitizenConfirm` **según `04` sin cambios** + nuestros códigos (`02` §10.10) + reglas de tolerancia al recibir; `/api/descargas/{token}` |
| Adaptadores GovCarpeta | F4, F6, F8 | DTOs del Swagger + traducción a dominio (una tabla por servicio dueño) |
| Máquinas de estado | F9 y F10 | Tablas de transición completas (estado, evento, acción, siguiente estado, reintento) derivadas de `02` §10.3 y §10.5 |

**Regla de cambio:** un contrato solo cambia con la aprobación del dueño y del consumidor, y se actualiza en `api-contracts.md` **antes** que el código.

---

## 8. Estrategia de pruebas

| Nivel | Dónde | Herramientas (stack aprobado) | Cubre |
|---|---|---|---|
| Unitarias | En cada servicio (`src/ms-*/src/test`) (H-06) | JUnit 5 | Reglas puras: correo, traducciones, validador de URLs, retroceso, transiciones |
| Componente | En cada servicio | `@QuarkusTest` + RestAssured + contenedores de BD y MinIO (Dev Services) + **WireMock** para GovCarpeta y operadores pares | Endpoints con dependencias reales locales y externas simuladas |
| Contrato | En cada servicio | Pruebas que comparan contra ejemplos de `api-contracts.md` | Que proveedor y consumidor no diverjan |
| Integración | `tests/integration/` | Scripts contra el stack de Compose | Recorridos completos, traslados entre dos instancias |
| Fallas y aislamiento | `tests/integration/fallas/` | Scripts de `docker compose stop/start` + verificaciones | Matriz §17.1 |
| Seguridad | Componente + integración | — | A↛B, sin token, token alterado, URLs de capacidad vencidas o alteradas, `/internal` no expuesto, confirmación falsa, validación de URLs |
| Frontend | `src/frontend` | Compilación + lint + lista de verificación manual | Pantallas y manejo de errores por sección |
| Humo real | Manual y documentada | GovCarpeta real + ids de prueba + limpieza | F4b, F6, F11 |

**Tokens para pruebas automáticas [APROBAR H-07]:** cliente `carpeta-tests` con *direct access grant* **solo en el realm de desarrollo o pruebas**, deshabilitado o ausente en el realm de demo.

---

## 9. Datos y configuración necesarios

| Grupo | Elementos | Fuente / responsable |
|---|---|---|
| Operador | `OPERATOR_ID`, `OPERATOR_NAME`, dirección, correo de contacto, participantes | H-02 + ejecución única de `registerOperator` |
| Público | `PUBLIC_BASE_URL` (hostname estable) + token o credencial del túnel | H-01 |
| GovCarpeta | `GOVCARPETA_BASE_URL`, timeouts | Swagger |
| Keycloak | URL interna, issuer público, realm `carpeta`, secretos de `ms-ciudadano` y `ms-interoperabilidad`, usuario administrador (solo local) | F1 |
| BD | Credenciales distintas para `pg-ciudadano`, `pg-documentos`, `pg-interop` | `.env` |
| MinIO | Credenciales (solo `ms-documentos`), bucket | `.env` |
| Parámetros | TTL de enlaces (15 / 60 min), plazo de confirmación (30 min), reintentos, límites (20 MB, 50 documentos), periodo del motor (10 s) | `02` §10.7 |
| Pruebas | Rango de ids ficticios (H-10); archivos de muestra (PDF, JPG, PNG válidos; >20 MB; ejecutable renombrado) | F0 |

Todos los secretos van en `.env`, que no se versiona. `.env.example` lleva solo nombres y valores ficticios.

---

## 10. Trabajo en paralelo

| Momento | Pistas simultáneas | Condición |
|---|---|---|
| Tras F0 | **A** F1 (plataforma); **B** preparación de proyectos y lógica pura de ms-ciudadano; **C** preparación de proyectos y lógica pura de ms-documentos; **D** esqueleto del frontend contra datos simulados que siguen `api-contracts.md` | Los contratos de F0 deben estar cerrados |
| Tras F1 | **A** F2 (frontera + spikes); **B** F4; **C** F5; **D** F3 → pantallas de F4 y F5 | Cada pista trabaja solo en su servicio y su BD |
| Tras F4 + F5 | **B** F7 (ciudadano); **C** F6 → F7 (documentos); **E** F8 (interop) contra WireMock de las APIs internas | Contratos `/internal/*` ya cerrados |
| Tras F7 + F8 | **E1** F9 (entrante); **E2** F10 (saliente); **D** pantallas de traslado | El motor de F8 es común: no se duplica |
| Secuencial obligatorio | F0 → cualquier integración; realm de Keycloak (F1) → pruebas con JWT; F5 → F6; F7 + F8 → F9 / F10; F9 + F10 + F2 → F11; F11 → F12 → F13 | — |

**Salvaguardas contra un paralelismo dañino:** cada pista modifica solo su carpeta `src/<servicio>`; no hay módulo de código compartido entre servicios (se tolera duplicar configuración trivial, como el formato de error, a cambio de independencia); ningún servicio recibe credenciales de otra BD ni de MinIO; los cambios de contrato pasan por §7.

---

## 11. Riesgos técnicos

| ID | Riesgo | Impacto | Mitigación |
|---|---|---|---|
| RT-01 | GovCarpeta no descarga o no acepta nuestra URL de capacidad | Cambia el diseño de acceso a documentos | SP-01 en F2, antes de F6 |
| RT-02 | Otros equipos no cumplen `04` igual (códigos, `confirmAPI`, formato de `urlDocuments`) | Traslados fallidos con terceros | Recepción tolerante; SP-06; F11b con instancias propias antes de F11c |
| RT-03 | El cliente del otro equipo descarta el `?t=` de `confirmAPI` | Nuestras confirmaciones serían rechazadas | Política de respaldo definida tras SP-06 (validación contra el estado + afiliación en GovCarpeta) |
| RT-04 | Hostname inestable u operador apagado cuando otros prueban | Interoperabilidad no demostrable | H-01 (estable); ventanas de prueba acordadas; volver a ejecutar `registerTransferEndPoint` |
| RT-05 | Complejidad de las máquinas de estado | Errores sutiles en la purga o la compensación | Tablas de transición cerradas en F0; una prueba por transición; motor común |
| RT-06 | Issuer de Keycloak detrás de NGINX y del túnel | Tokens rechazados | SP-08 en F2 |
| RT-07 | GovCarpeta (Heroku) lento, caído o compartido sin autenticación | Pruebas reales frágiles; colisión de ids | WireMock por defecto; rango de ids propio; limpieza |
| RT-08 | `registerOperator` es irreversible y su esquema es inconsistente | Registro mal hecho en el directorio compartido | H-02 + SP-07, ejecución única y controlada |
| RT-09 | Recursos de la máquina (≈10 contenedores, 3 JVM + Keycloak) y dos instancias en F11b | Lentitud o caídas | Límites de memoria por contenedor; F11b en una máquina con recursos suficientes |
| RT-10 | Herramientas del equipo (Node 14, sin Maven) | No se puede compilar en el host | Compilación en Docker (H-09) |
| RT-11 | Disponibilidad y licencia de las imágenes comunitarias de MinIO (su distribución cambió recientemente) | Imagen no disponible o sin actualizaciones | Verificar en F1 y fijar una versión conocida. Si no hay una imagen viable, escalar (sin cambiar el principio de almacenamiento S3 privado) |
| RT-12 | Carrera entre la confirmación y la respuesta al POST de traslado | Confirmación rechazada por error | `ENVIANDO` persistido antes del POST (F10) |
| RT-13 | Ciudadanos recibidos sin forma segura de activar su cuenta | Cuenta inaccesible o apropiable | H-03 |

---

## 12. ADRs a redactar (no creados todavía)

Los archivos ADR-001 a 005 existen **vacíos**. Propuesta de contenido cuando se aprueben (una decisión por archivo, siguiendo `02` §22.2):

| Archivo | Decisión (referencia en `02`) |
|---|---|
| ADR-001-microservices | Tres microservicios de negocio y sus límites (DA-01, §5) |
| ADR-002-public-boundary | NGINX como frontera de infraestructura con allowlist; hostname estable (DA-02, DA-03) |
| ADR-003-identity | Keycloak OIDC + PKCE, validación independiente, `citizen_id` (DA-04, DA-05, DA-10, DA-11) |
| ADR-004-document-access | MinIO privado + URLs de capacidad (DA-13, DA-14) |
| ADR-005-transfercitizen | Máquinas de estado, idempotencia, purga solo con confirmación, token en `confirmAPI` (DA-15 a DA-20, DA-23) |
| *Nuevos propuestos* | ADR-006 GovCarpeta por dominio (DA-06); ADR-007 comunicación híbrida sin broker (DA-17); ADR-008 persistencia por servicio (DA-07, DA-08, DA-09); ADR-009 simplificaciones frente a la entrega 2 (DA-25); ADR-010 health checks y aislamiento (DA-24); ADR-011 alcance práctico de seguridad (§3.2 de este plan) |

---

## 13. Decisiones que requieren aprobación antes de implementar

| ID | Decisión | Opciones | Recomendación | Bloquea |
|---|---|---|---|---|
| **H-01** | Mecanismo del hostname estable | (a) Túnel con nombre de Cloudflare + dominio propio; (b) dominio estático gratuito de ngrok; (c) VM pública | (a) si el equipo tiene dominio; si no, (b). En ambos casos el cliente corre **como contenedor** (sin instalación en el host) | F2, F6 real, F11 |
| **H-02** | Datos del operador (`OPERATOR_NAME` único y distintivo, dirección, correo, participantes) y responsable de ejecutar `registerOperator` | — | Definir y ejecutar una sola vez tras SP-07 | F4b, F11 |
| **H-03** | Activación de ciudadanos recibidos (`02` DA-22) | (a) Correo de Keycloak "definir contraseña" + **Mailpit** (contenedor nuevo, solo SMTP de desarrollo); (b) pantalla de activación con documento + correo (débil) | (a) | Cierre de F9 |
| **H-04** | Token de servicio en `/internal/*` (`02` DA-21) | (a) Red + token (client credentials); (b) solo aislamiento de red | (a): Quarkus lo resuelve con configuración, y protege operaciones de purga | F7, F8 |
| **H-05** | Cómo probar traslados entre dos operadores | (a) GovCarpeta **simulado con estado** (componente de prueba en `tests/`, en Java, no un servicio del producto) + dos instancias propias; (b) registrar un segundo operador real en GovCarpeta; (c) ambos | (c): (a) para pruebas repetibles y (b) una vez para la demo | F11 |
| **H-06** | Ubicación de las pruebas | (a) Unitarias y de componente dentro de cada servicio; `tests/` para caja negra; (b) todo en `tests/` | (a) | F4 |
| **H-07** | Cliente `carpeta-tests` con direct grant solo en desarrollo | Sí / no | Sí, excluido del realm de demo | Pruebas automáticas desde F4 |
| **H-08** | Crear el repositorio git y la estrategia de ramas | — | Rama principal + ramas por fase o pista | Paralelismo |
| **H-09** | Compilación | (a) Todo en Docker multi-stage + Maven Wrapper; (b) instalar Maven y Node 20 en los hosts | (a); (b) opcional para desarrollo con recarga en caliente | F1 |
| **H-10** | Rango de ids ficticios para GovCarpeta | — | Un rango propio documentado + limpieza tras cada sesión | F4b |
| **H-11** | Aceptar que la contraseña transite por `ms-ciudadano` (`02` DE-06) | Sí / no | Sí, sin persistirla ni registrarla en logs | F4 |
| **H-12** | Interpretación de "pendiente/reintentar" ante una caída de GovCarpeta (I-06) | Ver I-06 | Aceptar la interpretación | F4, F6 |
| **H-13** | Actualizar el estado de `02-arquitectura-propuesta.md` a "Aprobada" y anotar las precisiones de §3.2 | Sí / no | Sí | Claridad documental |
| H-14 | Confirmar el stack (Java 21 + Quarkus 3; React + TS) | — | Mantener (`02`) | F1 |

**Siguen abiertas con el profesor** (`02` §22.3). No bloquean el inicio, pero sí la demo:

- Q3: disponibilidad pública esperada.
- Q4: escenarios de falla que evaluará.
- Q5: esquema de `registerOperator` (SP-07 puede resolverla empíricamente).

---

## Anexo A. Propuesta de contenido para `CLAUDE.md` (NO aplicada)

```markdown
# Carpeta Ciudadana — Operador (guía para asistentes de código)

## Fuentes de verdad (orden)
1. resources/05-enunciado-implementacion.md (requisitos del profesor)
2. resources/01-caso-estudio.pdf
3. resources/04-transfer-citizen.md (contrato entre operadores; NO modificar el contrato)
4. docs/analysis/02-arquitectura-propuesta.md (arquitectura aprobada) + precisiones en docs/implementation/implementation-plan.md §3.2
5. docs/implementation/api-contracts.md (contratos; se actualizan ANTES que el código)
resources/ es SOLO LECTURA.

## Reglas de arquitectura (no cambiar sin aprobación humana)
- Tres microservicios: ms-ciudadano, ms-documentos, ms-interoperabilidad. Nada de microservicios nuevos ni monolito.
- Cada servicio: su propia PostgreSQL; nunca acceder a la BD, el almacenamiento o el código de otro servicio. Sin módulo compartido.
- Dependencias internas permitidas: SOLO ms-interoperabilidad → ms-ciudadano y ms-interoperabilidad → ms-documentos (APIs /internal/*).
- GovCarpeta: validate/register/unregisterCitizen solo en ms-ciudadano; authenticateDocument solo en ms-documentos; getOperators solo en ms-interoperabilidad.
- MinIO privado; solo ms-documentos lo usa. Acceso externo a documentos solo vía /api/descargas/{token}.
- Identidad: Keycloak; cada servicio valida el JWT; identificador de negocio = claim citizen_id (nunca sub).
- NGINX = infraestructura (routing, límites, request id, allowlist). Sin lógica de negocio. Solo NGINX publica puertos.
- Health: readiness solo de dependencias locales; nunca de otros microservicios ni de GovCarpeta.
- TransferCitizen: nunca purgar sin confirmación req_status=1 válida; nunca borrar por timeout.
- Prohibido sin aprobación: Kubernetes, Kong, RabbitMQ/Kafka, Vault, service mesh, MFA, observabilidad empresarial, BD compartidas.

## Forma de trabajo
- Implementar por fases de docs/implementation/implementation-plan.md.
- Ante un conflicto entre fuentes o una necesidad de cambio arquitectónico: reportar y DETENERSE.
- Stack: Java 21 + Quarkus 3 (Maven Wrapper), React + TypeScript, Docker Compose.
```
