# Guía de implementación de módulos backend

> **Propósito:** estándar técnico reutilizable para analizar, diseñar, implementar, probar y mantener módulos backend y sus endpoints. Esta guía consolida dos documentos de referencia y distingue los patrones observados en un proyecto FastAPI/SQLAlchemy con tablas legacy de las recomendaciones generales añadidas para completar el estándar.

## Tabla de contenido

1. [Introducción](#1-introducción)
2. [Principios generales de diseño backend](#2-principios-generales-de-diseño-backend)
3. [Análisis previo a la implementación](#3-análisis-previo-a-la-implementación-de-un-módulo)
4. [Diseño de endpoints](#4-diseño-de-endpoints)
5. [Operaciones CRUD](#5-operaciones-crud)
6. [Listados, paginación, filtros y búsquedas](#6-listados-paginación-filtros-y-búsquedas)
7. [Relaciones entre entidades](#7-relaciones-entre-entidades)
8. [Autenticación y autorización](#8-autenticación-y-autorización)
9. [Validaciones y reglas de negocio](#9-validaciones-y-reglas-de-negocio)
10. [Manejo de errores y respuestas HTTP](#10-manejo-de-errores-y-respuestas-http)
11. [Arquitectura y organización del código](#11-arquitectura-y-organización-del-código)
12. [Base de datos, transacciones e integridad](#12-base-de-datos-transacciones-e-integridad)
13. [Seguridad y buenas prácticas](#13-seguridad-y-buenas-prácticas)
14. [Pruebas](#14-pruebas)
15. [Documentación y mantenimiento](#15-documentación-y-mantenimiento)
16. [Flujo de trabajo recomendado](#16-flujo-de-trabajo-recomendado)
17. [Checklist de revisión](#17-checklist-de-revisión-de-un-módulo)
18. [Ejemplo práctico](#18-ejemplo-práctico-de-aplicación-de-la-guía)
19. [Glosario](#19-glosario)
20. [Referencias y trazabilidad](#20-referencias-y-trazabilidad)

---

## 1. Introducción

### 1.1 Propósito y objetivo

Crear un módulo backend no consiste únicamente en crear una tabla y exponer un CRUD. También exige definir su contrato HTTP, las reglas de negocio, sus dependencias, los permisos, el ámbito de los recursos, la integridad de los datos, el comportamiento ante errores y la forma de verificarlo.

Esta guía propone un proceso repetible para que el equipo pueda implementar módulos como usuarios, productos, publicaciones, comentarios, categorías, pedidos o catálogos sin asumir que todos necesitan las mismas operaciones o reglas.

### 1.2 Alcance y audiencia

Está dirigida a desarrolladores backend, revisores de código y responsables técnicos. Los ejemplos usan REST, SQL y principalmente Python con FastAPI/Pydantic/SQLAlchemy; algunos patrones son aplicables también a TypeScript con Node.js, Express o NestJS.

No sustituye los requisitos del producto, el esquema real de la base de datos ni las convenciones ya establecidas en el repositorio. Cuando se trabaja sobre un sistema existente, primero se debe verificar el código ejecutable y el contrato real: no se deben copiar comentarios obsoletos ni asumir que una guía específica describe perfectamente la implementación actual.

### 1.3 Cómo utilizar la guía

1. Analiza los requisitos y la base de datos antes de programar.
2. Define las operaciones, sus contratos y sus políticas de acceso.
3. Implementa por capas siguiendo la arquitectura real del proyecto.
4. Prueba casos positivos, negativos, de seguridad y de consistencia.
5. Documenta decisiones y completa el checklist de la sección 17.

### 1.4 Terminología

| Término | Significado en esta guía |
|---|---|
| `Module` | Unidad funcional que agrupa operaciones de un dominio. |
| `Resource` | Recurso principal administrado por el módulo. |
| `User` / actor | Identidad autenticada que realiza una operación. |
| `ParentResource` | Entidad padre de la que depende el recurso. |
| `RelatedResource` | Otra entidad relacionada. |
| `Permission` | Capacidad explícita para ejecutar una operación. |
| DTO | Objeto que define los datos aceptados o devueltos por una operación. |
| Caso de uso | Coordinación de una operación de aplicación y sus reglas. |
| Puerto/repositorio | Contrato de acceso a datos que desacopla la aplicación de la persistencia. |
| Tabla legacy | Tabla preexistente cuya estructura o ciclo de vida no está completamente controlado por el módulo nuevo. |

Las palabras **debe** y **obligatorio** indican un control necesario para cumplir el contrato o proteger los datos. **Recomendado** indica una buena práctica habitual. **Depende del contexto** señala una decisión que debe justificarse según el negocio, la arquitectura o la tecnología.

## 2. Principios generales de diseño backend

1. **Separación de responsabilidades.** La interfaz HTTP traduce solicitudes; la aplicación coordina casos de uso; el dominio expresa reglas propias; la infraestructura implementa persistencia y servicios externos.
2. **Bajo acoplamiento y alta cohesión.** Un módulo debe concentrar capacidades relacionadas y depender de contratos explícitos, no importar indiscriminadamente la implementación interna de otros módulos.
3. **Validación en los límites.** Validar tipos, estructura, longitudes y parámetros al recibir datos externos. Las reglas del negocio deben comprobarse también en la capa responsable, sin confiar en el frontend.
4. **Seguridad desde el diseño.** Cada operación protegida define autenticación, permiso, ámbito del recurso y campos modificables antes de implementarse.
5. **Integridad en más de una capa.** Las reglas de negocio viven en dominio/aplicación; las restricciones de persistencia protegen la consistencia frente a errores y concurrencia. Una FK no reemplaza la autorización.
6. **Contratos estables.** Las rutas, los DTO, los códigos HTTP, los errores y la paginación deben ser predecibles y documentados.
7. **Mantenibilidad.** Reutilizar políticas y componentes transversales existentes, pero evitar abstracciones genéricas que oculten reglas específicas.
8. **Eficiencia medible.** Definir límites de resultados y tamaño de petición, revisar consultas e índices según patrones de uso reales.
9. **Pruebas reproducibles.** No considerar terminado un módulo porque el caso feliz funciona; probar también permisos, límites, relaciones, errores y concurrencia relevante.

## 3. Análisis previo a la implementación de un módulo

No empieces por el router. Primero produce un pequeño diseño funcional y técnico que responda a lo siguiente.

### 3.1 Lista de análisis

- [ ] **Requisitos funcionales:** qué puede hacer el usuario y qué operaciones realmente necesita el módulo.
- [ ] **Reglas de negocio:** condiciones de creación, edición, eliminación, estados y transiciones permitidas.
- [ ] **Entidades y campos:** identificadores, tipos, campos obligatorios, valores por defecto, auditoría y datos sensibles.
- [ ] **Esquema real:** revisar DDL/migraciones y datos existentes; confirmar nombres, longitudes, tipos, nulabilidad y generación de IDs.
- [ ] **Claves y restricciones:** PK, FK, `UNIQUE`, `NOT NULL`, `CHECK` y restricciones compuestas.
- [ ] **Relaciones:** uno a uno (1:1), uno a muchos (1:N), muchos a muchos (N:M) y auto-referencias.
- [ ] **Dependencias entre módulos:** qué servicios/puertos necesita consultar o modificar el nuevo módulo.
- [ ] **Roles y permisos:** operaciones que requieren autenticación y permisos concretos.
- [ ] **Propiedad y ámbito:** si el acceso depende de propietario, organización, empresa, región, rol u otra condición.
- [ ] **Auditoría:** acciones y cambios que deben dejar trazabilidad.
- [ ] **Rendimiento:** volumen estimado, filtros frecuentes, ordenamientos, tamaño de respuesta e índices necesarios.
- [ ] **Transacciones:** escrituras que deben confirmarse o revertirse juntas.
- [ ] **Eliminación:** física, lógica, restrictiva o en cascada; impacto en entidades dependientes.
- [ ] **Contrato actual:** formato de listado, estilo de errores, códigos HTTP, versionado y convenciones del proyecto.
- [ ] **Migraciones y entornos:** cómo se crea o actualiza el esquema en desarrollo, pruebas y producción.

### 3.2 Inspección de una base existente

En un proyecto con una base legacy, inspecciona el esquema y algunas filas antes de modelar la tabla. Por ejemplo, con SQLite:

```bash
sqlite3 test_app.db ".schema SI_ESTADOS"
sqlite3 test_app.db "SELECT * FROM SI_ESTADOS LIMIT 5; SELECT COUNT(*) FROM SI_ESTADOS;"
```

Los comandos son ilustrativos: usa las herramientas del motor real. Compara la definición física con los modelos ORM y las migraciones. No des por hecho que los IDs son autoincrementales ni que todas las tablas legacy tienen claves foráneas declaradas.

### 3.3 Decisiones que deben quedar registradas

Como mínimo, registra: operaciones incluidas/excluidas, entidades relacionadas, permisos por operación, política de propiedad/ámbito, contrato de listado, estrategia de IDs, política de borrado, mapeo de errores, estrategia de pruebas y cualquier desviación de los patrones del repositorio.

## 4. Diseño de endpoints

### 4.1 Principios REST

- Usa sustantivos y nombres consistentes para recursos; evita representar cada endpoint como una acción arbitraria.
- Usa métodos HTTP según su semántica.
- Los identificadores de recursos van normalmente en la ruta; filtros, paginación y ordenamiento van en query string; los datos modificables van en el cuerpo.
- No expongas directamente el esquema de la base de datos como contrato público.
- Los recursos anidados son útiles cuando la relación contextual es importante, por ejemplo `POST /posts/{post_id}/comments`. No anides rutas sin necesidad.
- Mantén consistentes las convenciones de barra final, nombres, singular/plural y envoltorios de respuesta.

### 4.2 Tabla de operaciones habituales

| Operación | Método y ruta conceptual | Propósito | Respuesta típica |
|---|---|---|---|
| Crear | `POST /resources` | Crear un recurso | `201 Created` con representación creada |
| Listar | `GET /resources` | Consultar colección, filtros y paginación | `200 OK` con elementos y metadatos |
| Consultar | `GET /resources/{id}` | Obtener un recurso | `200 OK`; `404` si no existe o se oculta por política |
| Actualizar parcialmente | `PATCH /resources/{id}` | Cambiar solo campos enviados | `200 OK` con recurso actualizado o `204` sin cuerpo |
| Reemplazar | `PUT /resources/{id}` | Reemplazar representación completa | Usar solo si el contrato define reemplazo completo |
| Eliminar | `DELETE /resources/{id}` | Eliminar o desactivar según política | `204 No Content`; `404` o `409` según el caso |
| Crear en contexto padre | `POST /parents/{parent_id}/resources` | Crear recurso relacionado con un padre | `201 Created` |
| Operación de negocio | `POST /resources/{id}/actions` o ruta equivalente | Ejecutar una acción no modelada como CRUD | Definida por el contrato y la semántica de la acción |

No todos los módulos deben implementar todas estas operaciones. En el manual de proyecto de referencia, la convención existente utiliza `PATCH` para actualizaciones y no introduce `PUT` sin una decisión expresa; una guía genérica no debe forzar `PUT` si contradice el contrato del proyecto.

### 4.3 Contrato por endpoint

Define antes de implementar:

- Método, ruta, versión y propósito.
- Parámetros de ruta y query, tipos y límites.
- DTO de entrada y campos que se ignoran o rechazan.
- DTO de salida y campos sensibles excluidos.
- Autenticación, permisos, propiedad y ámbito de acceso.
- Validaciones y reglas de negocio.
- Respuestas exitosas y errores esperados.
- Efectos secundarios, transacciones e idempotencia.
- Límites de payload, paginación y solicitudes.

### 4.4 Idempotencia, concurrencia y versionado

`GET`, `PUT` y `DELETE` deberían mantener la semántica idempotente que les corresponde; `POST` normalmente no es idempotente. Para operaciones financieras o sensibles donde un reintento pueda duplicar efectos, considera una clave de idempotencia y almacenamiento del resultado. Si dos clientes pueden editar el mismo recurso, define control de concurrencia (por ejemplo, versión o `ETag`/`If-Match`). Introduce `/v1` u otro versionado cuando exista una estrategia de compatibilidad; no lo añadas mecánicamente si el proyecto no lo usa.

### 4.5 Ejemplo de contrato

```http
POST /posts/{post_id}/comments
Authorization: Bearer <token>
Content-Type: application/json
```

```json
{ "content": "Excelente publicación" }
```

Respuesta ilustrativa:

```json
{
  "id": "comment-456",
  "post_id": "post-123",
  "author_id": "user-789",
  "content": "Excelente publicación",
  "created_at": "2026-10-15T12:00:00Z"
}
```

El `author_id` procede de la identidad autenticada, no del cuerpo enviado por el cliente.

## 5. Operaciones CRUD

El CRUD es un patrón, no un requisito universal. Implementa solo las operaciones necesarias para el dominio y el perfil de acceso.

### 5.1 Crear

1. Validar formato, tipos, longitud y campos permitidos.
2. Autenticar y autorizar al actor.
3. Derivar del contexto autenticado los campos que el cliente no puede decidir (por ejemplo, propietario).
4. Validar existencia y ámbito de entidades relacionadas.
5. Aplicar reglas de negocio y restricciones de unicidad.
6. Persistir en una transacción si la operación incluye varias escrituras.
7. Devolver `201 Created` con un DTO de salida; opcionalmente usar `Location` si forma parte del estándar.

### 5.2 Consultar por identificador

- Validar formato del ID.
- Consultar el recurso dentro del ámbito autorizado siempre que sea posible.
- Aplicar permiso de lectura y política de propiedad/organización.
- Devolver solo los campos permitidos.
- Definir si un recurso inaccesible devuelve `403` o se presenta como `404` para evitar revelar su existencia.

### 5.3 Listar

- Aplicar filtros de ámbito antes de paginar; no pagines una colección global para filtrar después en memoria.
- Validar y limitar `limit`/`page_size`, filtros y ordenamiento.
- Mantener la misma semántica de filtros entre la consulta de elementos y el conteo total.
- Establecer un orden determinista.
- Documentar si `total` representa todos los resultados filtrados o solo los elementos de la página.

### 5.4 Actualizar total o parcialmente

- Definir explícitamente si se usa `PATCH` parcial o `PUT` completo.
- Validar que el recurso existe y que el actor puede modificarlo.
- Permitir solo campos editables; no aceptar `id`, propietario, permisos, timestamps de creación ni campos internos salvo regla explícita.
- Volver a validar las relaciones si se cambia una FK o colección relacionada.
- En `PATCH`, distinguir **campo ausente** de **campo enviado como `null`**. En Pydantic v2 puede utilizarse `model_fields_set` cuando sea necesario permitir borrar un valor opcional.
- Aplicar reglas de transición de estado y control de concurrencia cuando corresponda.

### 5.5 Eliminar

- Definir si el borrado es físico, lógico o una desactivación.
- Comprobar permisos y dependencias.
- Si otros registros dependen del recurso, decidir entre impedirlo (`409 Conflict`), aplicar una cascada explícitamente aceptada o preservar la referencia.
- No usar `CASCADE` por defecto sin analizar pérdida de datos e historial.
- Responder `204 No Content` cuando la operación concluye sin cuerpo, si esa es la convención elegida.

## 6. Listados, paginación, filtros y búsquedas

### 6.1 Diferencias

- **Paginación:** divide resultados en bloques.
- **Búsqueda:** encuentra coincidencias con un término.
- **Filtros:** restringen por atributos, fechas, estado o relaciones.
- **Ordenamiento:** ordena por columnas explícitamente permitidas.

Ejemplo de solicitud:

```http
GET /posts/123/comments?page=1&page_size=20&sort_by=created_at&sort_order=desc&q=excelente
```

### 6.2 Estrategias de paginación

**Offset:** sencilla y apropiada para listados con páginas numeradas, pero los offsets grandes pueden ser costosos y las inserciones concurrentes pueden desplazar los resultados.

```sql
SELECT id, content, created_at
FROM comments
WHERE post_id = :post_id
ORDER BY created_at DESC, id DESC
LIMIT :page_size OFFSET :offset;
```

`offset = (page - 1) * page_size`.

**Cursor:** suele convenir en feeds, historiales o colecciones grandes. Requiere orden estable y un cursor coherente con los campos ordenados.

```sql
SELECT id, content, created_at
FROM comments
WHERE post_id = :post_id
  AND (created_at, id) < (:cursor_created_at, :cursor_id)
ORDER BY created_at DESC, id DESC
LIMIT :page_size;
```

La comparación de tuplas depende del motor; el ejemplo corresponde a PostgreSQL. En APIs públicas suele ser preferible un cursor opaco validado por el servidor.

### 6.3 Límites, respuesta y validación

Define valores por defecto y máximo según el coste real; por ejemplo, `limit=20` y `max_limit=100` son valores ilustrativos, no una regla universal. Rechaza o normaliza parámetros inválidos de acuerdo con el contrato.

Ejemplo de respuesta con offset:

```json
{
  "items": [
    { "id": "c-1", "content": "Buen aporte" },
    { "id": "c-2", "content": "Gracias" }
  ],
  "total": 42,
  "offset": 0,
  "limit": 20
}
```

El formato de referencia de un proyecto puede ser `{items,total,offset,limit}`; otro puede usar `{data,total}`. No mezcles envoltorios de manera accidental: identifica el estándar existente y documenta toda excepción.

### 6.4 Rendimiento y seguridad de consultas

- Usa consultas parametrizadas o el mecanismo seguro del ORM.
- Nunca concatenes texto de usuario directamente en SQL.
- Los parámetros de valores no protegen nombres de columnas dinámicos: mapea `sort_by` a una lista de columnas permitidas.
- Asegura que `list` y `count` utilicen los mismos filtros, para que `total` no contradiga los resultados.
- Revisa índices según filtros y ordenamientos frecuentes; por ejemplo, `(post_id, created_at, id)` puede ayudar a listar comentarios por publicación y fecha.
- Evalúa planes de ejecución y volúmenes reales antes de añadir índices indiscriminadamente.
- Para búsquedas de texto complejas, evalúa las capacidades del motor (por ejemplo, full-text search o `pg_trgm` en PostgreSQL).
- Evita consultas N+1, límites sin tope y combinaciones de filtros que generen trabajo excesivo.

## 7. Relaciones entre entidades

### 7.1 Tipos y comprobaciones

| Relación | Ejemplo | Aspectos que revisar |
|---|---|---|
| 1:1 | perfil y usuario | unicidad de la FK y ciclo de vida conjunto |
| 1:N | publicación y comentarios | FK obligatoria/opcional, borrado del padre y ámbito del hijo |
| N:M | roles y permisos | tabla de enlace, unicidad compuesta, lectura mediante JOIN y sincronización de enlaces |
| Auto-referencia | categoría padre/hija | padre existente, evitar ciclos y profundidad permitida |

### 7.2 FK simple

Antes de guardar una referencia recibida, comprueba que la entidad relacionada existe y que el actor puede asociarla. La FK de la base de datos evita referencias inexistentes si está declarada, pero no comprueba permisos ni pertenencia al ámbito.

```python
# Ejemplo conceptual de caso de uso; los nombres son ilustrativos.
parent = parent_repository.get(parent_id)
if parent is None:
    raise ParentResourceNotFound(parent_id)
if not access_policy.can_attach(actor, parent):
    raise ForbiddenOperation()
resource = repository.create(parent_id=parent.id, ...)
```

La traducción HTTP del ID inexistente en una ruta y de una FK inválida en el cuerpo debe ser consistente con el contrato. Una convención posible es `404` para el recurso solicitado que no existe y `400` o `422` para una referencia de entrada inválida; no existe una única elección universal, por lo que debe documentarse.

### 7.3 Relaciones N:M

Para una tabla de enlace, comprueba que todos los IDs existen y son asignables. La lectura puede usar JOIN; la escritura puede sincronizar relaciones mediante operaciones equivalentes a eliminar enlaces anteriores e insertar los nuevos dentro de una misma transacción. Evita duplicados con una restricción única compuesta cuando el esquema lo permita.

### 7.4 Auto-referencias y borrado

En jerarquías, verifica que el padre exista y que asignarlo no genere ciclos. Antes de borrar una entidad con dependencias, determina si la política es restricción (`409`), cascada controlada o conservación de referencias. Documenta cualquier excepción debida al esquema legacy.

### 7.5 Respuestas relacionadas y consultas eficientes

Elige entre devolver solo `related_id` o incluir un objeto relacionado. La respuesta embebida puede mejorar la experiencia del cliente, pero amplía el contrato y puede añadir consultas. Haz explícita la elección en el DTO y usa JOIN/eager loading o consultas agrupadas para evitar N+1. Nunca serialices automáticamente todo el modelo ORM.

### 7.6 Protección contra IDs manipulados

Cada identificador recibido debe considerarse no confiable. Comprueba que el recurso padre y el hijo están relacionados entre sí y dentro del ámbito permitido. No basta con validar que ambos IDs existen de forma independiente.

## 8. Autenticación y autorización

### 8.1 Diferencia esencial

- **Autenticación:** responde quién realiza la petición.
- **Autorización:** determina qué acciones puede ejecutar y sobre qué recursos.

Una petición protegida sigue conceptualmente este flujo: verificar sesión/token → identificar al actor → comprobar permiso de operación → comprobar ámbito/propiedad del recurso → aplicar reglas de negocio → persistir.

### 8.2 Sesiones y tokens

La aplicación puede utilizar sesiones de servidor, JWT u OAuth 2.0/OpenID Connect según su arquitectura; no es obligatorio usar todos. Para JWT, según el diseño, valida firma y algoritmo permitido, expiración (`exp`), emisor (`iss`), audiencia (`aud`), tipo de token y estado de sesión/revocación cuando corresponda. Un `user_id` dentro de un token no es confiable si no se verifica criptográficamente.

Si roles/permisos se incluyen como claims, define cómo se reflejan los cambios de autorización antes de que el token expire. No aceptes roles, contraseñas ni IDs del actor enviados en el cuerpo como prueba de identidad.

### 8.3 RBAC y permisos por operación

RBAC asigna permisos a roles y roles a usuarios. Los códigos de permisos son convenciones de la aplicación, no palabras reservadas del framework o SQL.

| Permiso conceptual | Capacidad |
|---|---|
| `resource:create` | Crear recursos |
| `resource:read` | Consultar recursos |
| `resource:update` | Actualizar recursos |
| `resource:delete` | Eliminar recursos |
| `resource:moderate` | Ejecutar acciones de moderación, si existen |

Tener acceso a un módulo no implica automáticamente poder realizar todas las operaciones. Define permisos por capacidad cuando el negocio requiera separación de acciones. En algunos sistemas legacy el guard verifica un slug general del módulo; esa convención debe documentarse y no debe confundirse con permisos CRUD separados.

### 8.4 Autorización a nivel de recurso

Un permiso como `resource:update` puede ser necesario pero no suficiente. Una política típica es:

```text
Puede actualizar = tiene resource:update
                   Y (es propietario O tiene permiso especial de administración)
                   Y el recurso pertenece al ámbito autorizado
```

El ámbito puede ser organización, empresa, región, propietario o una combinación. Define el comportamiento para administradores, usuarios normales, moderadores y actores sin rol. Los permisos de interfaz solo controlan la presentación; la validación efectiva debe ocurrir en el backend.

### 8.5 Guards y acceso a datos del actor

Elige el mecanismo existente según lo que el endpoint necesite: solo comprobar permisos; comprobar permiso y cargar el usuario; o cargar al usuario para aplicar scoping por rol/organización. No asumas que todos los guards hacen lo mismo ni que el framework aplica autenticación global automáticamente. En el manual de referencia, los routers nuevos sin dependencia explícita de guard quedan públicos.

### 8.6 401, 403 y 404

- `401 Unauthorized`: falta autenticación válida o el token/sesión no es válido.
- `403 Forbidden`: el actor está autenticado, pero no tiene permiso suficiente.
- `404 Not Found`: el recurso solicitado no existe; algunas políticas también ocultan con `404` los recursos inaccesibles.

La decisión de ocultar la existencia de un recurso debe ser deliberada y coherente, no una sustitución accidental de errores.

## 9. Validaciones y reglas de negocio

### 9.1 Capas de validación

| Tipo | Ejemplo | Dónde suele aplicarse |
|---|---|---|
| Sintáctica | JSON malformado, UUID inválido | Límite HTTP/framework |
| Estructural y de tipos | campo requerido, string máximo | DTO/schema de entrada |
| Semántica | referencia válida, fecha dentro de rango | aplicación/caso de uso y consultas necesarias |
| Negocio | publicación cerrada a comentarios | dominio/caso de uso |
| Autorización | actor no puede modificar recurso ajeno | guard/política/caso de uso |
| Integridad | unicidad, FK, `NOT NULL` | base de datos y manejo de conflictos |
| Transición de estado | no pasar de `closed` a `draft` | dominio/caso de uso |

No dupliques sin motivo una regla idéntica en todas las capas. Define dónde es autoritativa y usa la base de datos como última defensa para invariantes que puede garantizar. El esquema HTTP no reemplaza las reglas del dominio.

### 9.2 Ejemplo Pydantic v2

```python
from pydantic import BaseModel, Field, field_validator

class ResourceCreate(BaseModel):
    name: str = Field(min_length=1, max_length=100)

    @field_validator("name")
    @classmethod
    def name_not_blank(cls, value: str) -> str:
        if not value.strip():
            raise ValueError("El nombre no puede estar vacío")
        return value
```

Los límites son ilustrativos. Define la normalización (espacios, Unicode, mayúsculas) según las reglas del dominio. No elimines indiscriminadamente caracteres especiales: apóstrofos y símbolos pueden ser datos válidos.

### 9.3 Campos controlados por el servidor

El DTO de creación/actualización debe permitir explícitamente solo los campos modificables. No permitas asignación masiva de `author_id`, `owner_id`, `role_id`, permisos, `created_at` o flags internos salvo que exista un caso de uso autorizado para ello. Decide si los campos extra se rechazan o se ignoran y documenta la política.

## 10. Manejo de errores y respuestas HTTP

### 10.1 Códigos habituales

| Código | Uso habitual |
|---|---|
| `200 OK` | Lectura o actualización con cuerpo |
| `201 Created` | Creación exitosa |
| `204 No Content` | Operación exitosa sin cuerpo |
| `400 Bad Request` | Solicitud inválida según el contrato |
| `401 Unauthorized` | Autenticación ausente o inválida |
| `403 Forbidden` | Actor autenticado sin permiso |
| `404 Not Found` | Recurso inexistente o deliberadamente oculto |
| `409 Conflict` | Duplicado, recurso en uso o conflicto de estado |
| `422 Unprocessable Content` | Entrada que no supera validación semántica, si ese es el contrato |
| `429 Too Many Requests` | Se excedió un límite |
| `500 Internal Server Error` | Fallo inesperado |

FastAPI/Pydantic puede responder `422` automáticamente por errores de esquema. Un proyecto existente puede usar `400` para errores de dominio o referencias FK inválidas. No cambies la convención sin revisar clientes y documentación.

### 10.2 Formato de error uniforme

```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "No se encontró el recurso solicitado",
    "details": null,
    "request_id": "req-123"
  }
}
```

`details` puede contener errores de campo seguros y útiles; `request_id` facilita correlacionar la respuesta con los logs. El formato es un ejemplo: conserva el contrato existente si ya hay uno.

### 10.3 Mapeo de excepciones

Usa errores de dominio/aplicación con significado claro (por ejemplo, `ResourceNotFound`, `ResourceAlreadyExists`, `ResourceInUse`) y tradúcelos en un punto central a HTTP. Evita que el dominio dependa de `HTTPException` o de objetos del framework. Asegura que cada error tenga un mapeo explícito y que el fallback no oculte excepciones mal clasificadas.

### 10.4 Errores inesperados

Registra internamente el detalle técnico necesario, pero responde con un mensaje genérico. No expongas stack traces, SQL, tokens, contraseñas, secretos ni detalles de infraestructura. Evita que los mensajes de errores de dominio filtren datos sensibles.

## 11. Arquitectura y organización del código

La arquitectura debe seguir la convención del proyecto. Si no existe una, la arquitectura hexagonal/limpia ofrece una separación útil para aplicaciones con reglas de negocio relevantes.

### 11.1 Estructura de referencia

```text
src/
├── domain/
│   ├── entities/
│   ├── ports/                 # interfaces de repositorio/servicios
│   └── errors.py              # errores de dominio
├── application/
│   ├── dto/
│   └── use_cases/             # casos de uso
├── infrastructure/
│   ├── database/              # modelos, sesiones y mappers
│   └── repositories/           # implementaciones de puertos
└── interfaces/
    └── api/
        ├── routers/
        ├── dependencies/       # inyección y providers
        ├── exception_handlers.py
        └── app.py
```

En proyectos modulares puede organizarse primero por módulo y después por capa. No es necesario crear una carpeta para cada concepto en un proyecto pequeño; la prioridad es que las responsabilidades y dependencias sean claras.

### 11.2 Responsabilidades

- **Domain:** entidades, invariantes y reglas esenciales; no importa FastAPI, SQLAlchemy ni detalles HTTP.
- **Application:** casos de uso, coordinación de políticas, puertos necesarios y DTO de aplicación cuando corresponda. No depende de objetos `Request` ni de ORM.
- **Repository port:** define operaciones y errores esperados; su contrato debe coincidir con lo que la aplicación necesita.
- **Infrastructure:** modelos ORM, consultas, mappers, transacciones, servicios externos y adaptadores que implementan los puertos.
- **Interfaces/router/controller:** parámetros HTTP, schemas de transporte, dependencias de autenticación, llamada al caso de uso y conversión de resultados a respuestas.
- **Composition root/providers:** construye y conecta las implementaciones concretas.
- **Exception handlers:** traducen errores conocidos al contrato HTTP.

### 11.3 Flujo de petición

```text
Cliente HTTP
  → middleware/guard de autenticación
  → autorización de operación y recurso
  → router/controller + validación HTTP
  → caso de uso
  → entidad/política de dominio y puertos
  → repositorio concreto
  → base de datos
  → mapper/DTO de salida
  → respuesta HTTP
```

El orden exacto puede variar: por ejemplo, la autorización a nivel de recurso puede requerir consultar datos dentro del caso de uso. Lo importante es no omitirla ni depender exclusivamente de un guard de ruta si el caso de uso se invoca desde otros puntos de entrada.

### 11.4 Entidad, DTO, mapper y repositorio

- La **entidad de dominio** representa conceptos y reglas del negocio; no debe ser un espejo automático de todas las columnas.
- El **DTO/schema** define la forma de entrada o salida.
- El **mapper** convierte entre modelo de persistencia y entidad/DTO cuando sea útil.
- El **puerto de repositorio** expresa lo que necesita la aplicación.
- La **implementación de repositorio** contiene consultas y detalles del ORM.
- Los **casos de uso** coordinan acciones y no deberían convertirse en una copia del router ni en una capa que solo reenvía llamadas sin aportar separación útil.

### 11.5 Dependencias entre módulos

Si un módulo necesita verificar una publicación o una región, utiliza un puerto/servicio de lectura o una interfaz explícita. Evita importar repositorios internos indiscriminadamente. Cuando varios módulos comparten una misma base, no asumas que eso elimina la necesidad de definir límites de responsabilidad.

## 12. Base de datos, transacciones e integridad

### 12.1 Restricciones y consultas

- Define PK, FK, `UNIQUE`, `NOT NULL` y `CHECK` donde el motor y el esquema lo permitan.
- Usa consultas parametrizadas o APIs seguras del ORM.
- Mantén tipos y longitudes de modelos sincronizados con el esquema real, especialmente en tablas legacy.
- Si el ID no es autogenerado, documenta el mecanismo. `MAX(id)+1` puede aparecer en sistemas legacy, pero es vulnerable a carreras si dos transacciones calculan el mismo valor; no lo adoptes como estrategia nueva sin un mecanismo de sincronización/garantía.
- Si una tabla legacy no tiene restricciones declaradas, verifica las reglas en aplicación y evalúa cómo protegerlas frente a concurrencia.

### 12.2 Transacciones

Usa una transacción cuando varias escrituras forman una única operación lógica, como crear un recurso y su registro de auditoría. Si una operación falla, revierte todas las escrituras relacionadas.

```text
BEGIN
  INSERT recurso
  INSERT auditoría
COMMIT
```

Si falla una escritura: `ROLLBACK`.

Una transacción SQL no revierte por sí misma correos, llamadas HTTP ni eventos externos. Para publicar eventos fiables junto con cambios de base de datos, evalúa patrones como Transactional Outbox.

### 12.3 Concurrencia y condiciones de carrera

Dos procesos pueden validar la misma condición y escribir al mismo tiempo. Las verificaciones de unicidad solo en código no garantizan ausencia de duplicados bajo concurrencia. Prefiere restricciones únicas en BD cuando sea posible y traduce el conflicto al error de negocio apropiado.

Para evitar actualizaciones perdidas, puede usarse una columna de versión:

```sql
UPDATE resources
SET name = :name, version = version + 1
WHERE id = :id AND version = :expected_version;
```

Si no se actualiza ninguna fila, determina si el recurso no existe o si la versión cambió. Alternativas incluyen bloqueo de filas y niveles de aislamiento adecuados. Las operaciones críticas también deben evitar que la autorización cambie entre la comprobación y la escritura; utiliza actualización condicionada, transacción o bloqueo según el caso.

### 12.4 Migraciones y tablas legacy

Toda modificación de esquema debe tener un proceso reproducible para los entornos necesarios. En tablas legacy, confirma quién es propietario del esquema y si las migraciones automáticas pueden alterarlas. El manual de referencia recomienda migraciones manuales e idempotentes para determinadas tablas `SI_*`/`USERS_*`, registrarlas en `LEGACY_TABLES` y evitar `--autogenerate` si el entorno de Alembic las excluye. **Esto es una regla específica de ese repositorio, no una regla universal de Alembic.**

Si una migración debe ser idempotente, comprueba existencia antes de crear. Un `downgrade` que no elimina una tabla puede ser deliberado para proteger datos preexistentes, pero debe justificarse y documentarse.

### 12.5 Eliminación física y lógica

Elige entre `RESTRICT`, `CASCADE`, `SET NULL`, borrado lógico o conservación histórica según las reglas del negocio. Revisa efectos secundarios en auditoría, reportes, claves foráneas y políticas de acceso. No supongas que todos los proyectos tienen soft delete: el manual analizado indica que el proyecto de referencia usa borrados físicos salvo casos concretos como la activación de usuarios.

## 13. Seguridad y buenas prácticas

### 13.1 Controles esenciales

| Riesgo | Control recomendado |
|---|---|
| BOLA/IDOR: cambiar un ID para acceder a recursos ajenos | Autorizar cada recurso y comprobar relación/ámbito, no solo el permiso general |
| Inyección SQL | Consultas parametrizadas y ORM usado correctamente |
| Asignación masiva | DTO con lista explícita de campos permitidos |
| Exposición excesiva | DTO de salida limitado; no serializar el modelo completo |
| Escalada de privilegios | No aceptar roles/permisos del cliente como autoridad |
| Robo de credenciales | Gestionar sesiones/tokens/secretos de forma segura |
| XSS por contenido almacenado | Codificación contextual en la salida y sanitización si se admite HTML |
| CSRF en cookies | Evaluar token CSRF y configurar `SameSite`, `Secure`, `HttpOnly` adecuadamente |
| Abuso/spam | Rate limiting, cuotas y controles antiabuso cuando corresponda |
| Consultas costosas | Límites de tamaño, resultados, filtros y concurrencia; revisar consultas e índices |
| Fuga en logs | No registrar contraseñas, tokens ni datos sensibles innecesarios |
| CORS mal configurado | Permitir orígenes necesarios; recordar que CORS no sustituye autorización |

### 13.2 OWASP API Security

La guía de OWASP API Security es una referencia externa relevante, especialmente para autorización a nivel de objeto (BOLA), autenticación, autorización a nivel de propiedad, consumo no limitado de recursos y configuración incorrecta. El control específico debe elegirse según el riesgo real del módulo y la arquitectura.

Referencia: [OWASP API Security](https://owasp.org/API-Security/).

### 13.3 Auditoría, logs y observabilidad

Distingue:

- **Logs técnicos:** excepciones, latencia, errores y consultas lentas.
- **Auditoría de negocio:** quién hizo qué, sobre qué recurso, cuándo y con qué resultado.
- **Métricas y trazas:** tasa de errores, tiempos, volumen y recorrido entre servicios.

Ejemplo de evento sin contenido sensible:

```text
event: resource.updated
actor_id: user-123
resource_id: resource-456
request_id: req-789
result: success
```

No es necesario auditar todas las lecturas en todos los sistemas. Define los eventos exigidos por negocio, cumplimiento y soporte. Registra accesos denegados cuando sea útil, con controles para que los intentos masivos no saturen los logs.

## 14. Pruebas

Un módulo debe probar el comportamiento funcional, el contrato HTTP y los límites de seguridad. Utiliza usuarios y roles distintos; probar solo con administrador puede ocultar errores.

### 14.1 Tipos

- **Unitarias:** reglas de dominio, políticas y casos de uso con repositorios falsos/mocks cuando proceda.
- **Integración:** repositorios, consultas, transacciones y restricciones contra una base de datos representativa.
- **API/contrato:** rutas, validación, serialización, códigos HTTP, autenticación y respuestas.
- **Seguridad:** permisos, propiedad, IDs manipulados, campos no permitidos y abuso de recursos.
- **Rendimiento:** consultas e índices con volúmenes realistas cuando el riesgo lo justifique.

### 14.2 Matriz de escenarios

| Área | Casos positivos | Casos negativos/límite |
|---|---|---|
| Autenticación | token/sesión válida | ausente, expirado, firma inválida, usuario deshabilitado |
| Permisos | actor con permiso | actor autenticado sin permiso; rol retirado; permiso no configurado |
| Propiedad/ámbito | propietario o actor autorizado | ID de otro usuario, otra organización/región o padre no relacionado |
| Validación | payload válido | campo requerido ausente, tipo incorrecto, vacío, demasiado largo, campo prohibido |
| Relaciones | padre y referencias válidas | padre inexistente, FK inválida, relación no permitida, duplicados |
| CRUD | crear, leer, actualizar y borrar autorizados | recurso inexistente, duplicado, recurso en uso, transición inválida |
| Listado | filtros, orden y páginas correctas | límites fuera de rango, sort no permitido, cursor inválido, página vacía |
| Persistencia | transacción completa | rollback, restricción violada, escrituras concurrentes |
| Seguridad | respuesta no filtra datos | inyección, exposición de campos, rate limit, errores internos filtrados |

### 14.3 Pruebas de autorización por objeto

Para cada operación sobre un recurso, crea al menos dos usuarios con recursos distintos. Verifica que el segundo usuario no pueda leer, editar ni borrar el recurso del primero cambiando el ID. Repite para permisos especiales de administración/moderación y para límites de organización o región.

### 14.4 Pruebas en proyectos existentes

Sigue las herramientas y fixtures del repositorio. El manual de referencia describe una suite con BD temporal recreada por test, fixtures de datos base, pruebas de integración de la app real y pruebas unitarias de casos de uso con repositorio falso. Esas decisiones son propias de ese proyecto; adapta la estrategia a la infraestructura de pruebas real.

## 15. Documentación y mantenimiento

- Documenta endpoints en OpenAPI/Swagger: método, ruta, propósito, autenticación, permisos, parámetros, DTO, respuestas y errores.
- Incluye ejemplos reales pero no sensibles de solicitudes y respuestas.
- Documenta el contrato de listados: envoltorio, significado de `total`, paginación, filtros y ordenamiento.
- Mantén actualizados códigos de permisos, handlers de excepciones y documentación de errores.
- Registra cambios incompatibles y define una estrategia de migración/versionado.
- Documenta decisiones arquitectónicas relevantes y excepciones al patrón.
- Mantén pruebas alineadas con el contrato; un cambio de status o forma de respuesta también es un cambio de API.
- Revisa el esquema y los comentarios de código contra la implementación real: el manual de referencia señala ejemplos de docstrings/guard comments obsoletos.
- Usa métricas y logs para detectar regresiones, latencia y errores en producción.

## 16. Flujo de trabajo recomendado

1. **Analizar requisitos:** define actores, escenarios, restricciones y criterios de aceptación.
2. **Inspeccionar el sistema:** lee convenciones, módulos similares, esquema real, migraciones y tests existentes.
3. **Identificar entidades y relaciones:** documenta PK/FK, cardinalidades, unicidad, nulabilidad y ciclo de vida.
4. **Definir reglas de negocio:** estados, propiedad, restricciones, duplicados, eliminación y concurrencia.
5. **Diseñar contrato HTTP:** rutas, métodos, parámetros, DTO, status y errores.
6. **Definir seguridad:** autenticación, permisos por operación y política de ámbito/propiedad.
7. **Diseñar persistencia:** consultas, transacciones, índices, IDs, migraciones y estrategia de integridad.
8. **Implementar dominio y puertos** según la arquitectura seleccionada.
9. **Implementar casos de uso/servicios de aplicación** y reglas de coordinación.
10. **Implementar repositorios y mappers** con consultas seguras y errores tipados.
11. **Implementar schemas, routers y DI**; registrar el router en el punto de composición.
12. **Configurar el mapeo de errores** y las respuestas uniformes.
13. **Crear pruebas unitarias, de integración, API y seguridad.**
14. **Documentar OpenAPI y decisiones** y verificar los ejemplos.
15. **Revisar seguridad, rendimiento, accesibilidad operativa y mantenibilidad.**
16. **Ejecutar verificaciones:** lint, tipado, suite de tests, migraciones y smoke tests.
17. **Completar checklist y criterios de aceptación** antes de integrar o desplegar.

El orden puede cambiar si una decisión de arquitectura o un requisito del proyecto lo exige; registra el motivo.

## 17. Checklist de revisión de un módulo

Marca cada fila como **Cumplido**, **No cumplido** o **No aplica**. Para uso en pull requests, se puede añadir una columna de estado y otra de evidencia.

| Categoría | Verificación | Estado |
|---|---|---|
| Análisis | Requisitos, actores y criterios de aceptación definidos | ☐ Cumplido / ☐ No cumplido / ☐ N/A |
| Análisis | Entidades, campos, cardinalidades y dependencias documentados | ☐ Cumplido / ☐ No cumplido / ☐ N/A |
| Análisis | Reglas de negocio y políticas de borrado definidas | ☐ Cumplido / ☐ No cumplido / ☐ N/A |
| API | Rutas, métodos y parámetros siguen el estándar del proyecto | ☐ Cumplido / ☐ No cumplido / ☐ N/A |
| API | DTO de entrada/salida y códigos HTTP documentados | ☐ Cumplido / ☐ No cumplido / ☐ N/A |
| API | Contrato de paginación, filtros y ordenamiento definido | ☐ Cumplido / ☐ No cumplido / ☐ N/A |
| Datos | Tipos, longitudes, IDs y nulabilidad coinciden con el esquema real | ☐ Cumplido / ☐ No cumplido / ☐ N/A |
| Datos | PK/FK/UNIQUE/CHECK e índices se han evaluado | ☐ Cumplido / ☐ No cumplido / ☐ N/A |
| Datos | Transacciones, concurrencia y condiciones de carrera evaluadas | ☐ Cumplido / ☐ No cumplido / ☐ N/A |
| Datos | Migraciones reproducibles y seguras para datos existentes | ☐ Cumplido / ☐ No cumplido / ☐ N/A |
| Seguridad | Cada endpoint tiene decisión explícita de autenticación | ☐ Cumplido / ☐ No cumplido / ☐ N/A |
| Seguridad | Permisos por operación y ámbito/propiedad comprobados en backend | ☐ Cumplido / ☐ No cumplido / ☐ N/A |
| Seguridad | Campos sensibles/internos no son asignables ni expuestos indebidamente | ☐ Cumplido / ☐ No cumplido / ☐ N/A |
| Seguridad | Consultas parametrizadas y límites antiabuso adecuados | ☐ Cumplido / ☐ No cumplido / ☐ N/A |
| Validación | Tipos, límites, campos extra y semántica de `null` están definidos | ☐ Cumplido / ☐ No cumplido / ☐ N/A |
| Validación | Reglas de negocio y transiciones de estado verificadas | ☐ Cumplido / ☐ No cumplido / ☐ N/A |
| Arquitectura | Router/controlador delega; responsabilidades y dependencias son claras | ☐ Cumplido / ☐ No cumplido / ☐ N/A |
| Arquitectura | Puertos, repositorios, mappers y errores están alineados | ☐ Cumplido / ☐ No cumplido / ☐ N/A |
| Pruebas | Casos felices, errores y límites están cubiertos | ☐ Cumplido / ☐ No cumplido / ☐ N/A |
| Pruebas | Autorización, propiedad e IDs manipulados tienen pruebas | ☐ Cumplido / ☐ No cumplido / ☐ N/A |
| Pruebas | Consultas, transacciones y concurrencia se prueban cuando aplica | ☐ Cumplido / ☐ No cumplido / ☐ N/A |
| Documentación | OpenAPI y ejemplos reflejan la implementación | ☐ Cumplido / ☐ No cumplido / ☐ N/A |
| Documentación | Permisos, errores y decisiones especiales están documentados | ☐ Cumplido / ☐ No cumplido / ☐ N/A |
| Operación | Logs, request/correlation ID y auditoría requeridos están definidos | ☐ Cumplido / ☐ No cumplido / ☐ N/A |
| Operación | Lint, tipado, pruebas y smoke checks pasan | ☐ Cumplido / ☐ No cumplido / ☐ N/A |

**Criterio de salida:** no debe quedar ningún control de seguridad obligatorio como “No cumplido”. Los elementos “No aplica” deben tener una justificación cuando afecten al diseño o al riesgo.

## 18. Ejemplo práctico de aplicación de la guía

Este ejemplo utiliza `comments` relacionado con `posts` y `users`. Las reglas concretas son ilustrativas y deben confirmarse con el negocio.

### 18.1 Descripción y modelo

Un usuario autenticado puede crear un comentario en una publicación abierta. El autor puede leer y editar sus comentarios; un moderador con permiso específico puede intervenir sobre comentarios ajenos. El comentario tiene un ID, publicación, autor, contenido y timestamps.

```sql
CREATE TABLE comments (
    id UUID PRIMARY KEY,
    post_id UUID NOT NULL REFERENCES posts(id),
    author_id UUID NOT NULL REFERENCES users(id),
    content TEXT NOT NULL CHECK (length(trim(content)) > 0),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_comments_post_created_id
    ON comments (post_id, created_at DESC, id DESC);
```

El `CHECK` y el índice son ilustrativos. Ajusta el soporte de expresiones y las reglas al motor y a la semántica requerida.

### 18.2 Reglas de negocio

1. El actor debe estar autenticado.
2. El actor debe tener `comments:create` para crear y `comments:read` para consultar.
3. Solo el propietario o un actor con permiso de moderación puede editar/eliminar.
4. La publicación debe existir y aceptar comentarios.
5. `author_id` se obtiene del actor autenticado.
6. El contenido no puede estar vacío y tiene un máximo definido por el producto.
7. La eliminación de la publicación debe tener política explícita respecto a sus comentarios.

### 18.3 Endpoints y permisos

| Método y ruta | Permiso conceptual | Resultado principal |
|---|---|---|
| `POST /posts/{post_id}/comments` | `comments:create` | `201` |
| `GET /posts/{post_id}/comments` | `comments:read` | `200` + paginación |
| `GET /comments/{comment_id}` | `comments:read` + ámbito | `200` o `404` |
| `PATCH /comments/{comment_id}` | `comments:update` + propietario/moderador | `200` |
| `DELETE /comments/{comment_id}` | `comments:delete` + propietario/moderador | `204` |

El permiso general y la propiedad se comprueban por separado. Si la aplicación usa un guard legacy de acceso a módulo, documenta la diferencia y verifica que las restricciones por recurso sigan aplicándose.

### 18.4 Ejemplo de caso de uso

```python
class CreateCommentUseCase:
    def __init__(self, comments, posts, access_policy):
        self.comments = comments
        self.posts = posts
        self.access_policy = access_policy

    def execute(self, *, actor, post_id, content):
        post = self.posts.get(post_id)
        if post is None:
            raise PostNotFound(post_id)
        if not self.access_policy.can_comment(actor, post):
            raise ForbiddenOperation()
        if not post.comments_open:
            raise CommentsClosed(post_id)
        if not content.strip():
            raise InvalidCommentContent()
        return self.comments.create(
            post_id=post_id,
            author_id=actor.id,
            content=content,
        )
```

Es pseudocódigo arquitectónico: los tipos, nombres de excepciones y métodos dependen del proyecto. El repositorio debe persistir de forma segura y la capa HTTP debe mapear errores conocidos al contrato de la API.

### 18.5 Respuestas y errores

Creación exitosa (`201`):

```json
{
  "id": "comment-456",
  "post_id": "post-123",
  "author_id": "user-789",
  "content": "Excelente publicación",
  "created_at": "2026-10-15T12:00:00Z"
}
```

Error de validación (ejemplo de contrato `422`):

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "La solicitud contiene datos inválidos",
    "details": [{ "field": "content", "reason": "must_not_be_blank" }],
    "request_id": "req-123"
  }
}
```

Los códigos exactos deben alinearse con el estándar de la aplicación; el ejemplo no obliga a sustituir un contrato existente.

### 18.6 Casos de prueba

- `POST` con actor autorizado, publicación abierta y contenido válido → `201`.
- Sin token o token inválido → `401`.
- Actor sin permiso → `403`.
- Publicación inexistente → error documentado de recurso/referencia inexistente.
- Publicación cerrada → conflicto/regla de negocio según el contrato.
- Contenido vacío o demasiado largo → error de validación.
- Cliente intenta enviar `author_id` de otra persona → campo rechazado/ignorado según política; nunca cambia el autor efectivo.
- Usuario A intenta leer/editar/borrar comentario de B cambiando el ID → acceso denegado o `404` deliberado.
- Comentario actualizado simultáneamente por dos actores → se aplica la política de concurrencia definida.
- Listado con filtros y paginación → `items` y `total` coinciden con el mismo conjunto filtrado.
- Error al guardar comentario o auditoría → rollback de todas las escrituras que forman la misma transacción.

### 18.7 Qué es general y qué es específico

**Generalizable:** separar permiso de operación y autorización del recurso; validar el padre; derivar el actor del contexto autenticado; limitar listados; usar DTO; probar IDs manipulados.

**Específico del ejemplo:** nombres `comments/posts/users`, permisos `comments:*`, política de moderación, apertura de comentarios y columnas mostradas.

## 19. Glosario

| Término | Definición |
|---|---|
| API REST | Interfaz HTTP orientada a recursos y contratos explícitos. |
| ABAC | Control de acceso basado en atributos del actor, recurso o contexto. |
| RBAC | Control de acceso basado en roles. |
| DTO | Objeto que delimita los datos de entrada o salida de una operación. |
| FK | Clave foránea que referencia un registro relacionado. |
| IDOR | Referencia directa a objeto que permite acceso indebido cuando falta autorización. |
| BOLA | Fallo de autorización a nivel de objeto; el usuario puede acceder a objetos que no le corresponden. |
| CRUD | Create, Read, Update, Delete: crear, leer, actualizar y eliminar. |
| Idempotencia | Propiedad por la que repetir una operación produce el mismo efecto final previsto. |
| Migración | Cambio versionado y reproducible del esquema o los datos. |
| N+1 | Patrón ineficiente donde se ejecuta una consulta inicial y luego una consulta adicional por cada registro. |
| OpenAPI | Especificación para describir contratos de APIs HTTP. |
| Puerto | Contrato que define una capacidad requerida/provista sin acoplarla a una implementación concreta. |
| Repository | Abstracción que encapsula consultas y persistencia. |
| Transacción | Grupo de operaciones que se confirman o revierten como unidad. |
| Concurrencia optimista | Estrategia que detecta cambios concurrentes mediante versión/condición en vez de bloquear siempre. |
| Soft delete | Marca un registro como eliminado sin borrarlo físicamente. |
| Outbox transaccional | Patrón para guardar un evento en la misma transacción que los cambios de datos y publicarlo después. |

## 20. Referencias y trazabilidad

### 20.1 Documentos fuente analizados

1. **`guia_diseno_modulos_backend(1).md`** — *Guía de ingeniería backend: diseño de módulos y endpoints seguros*. Aporta el enfoque general de modelo de datos, REST, autenticación JWT/sesiones, RBAC y propiedad, validación por capas, paginación offset/cursor, errores HTTP, transacciones, concurrencia, seguridad, DTO, arquitectura hexagonal, pruebas y observabilidad.
2. **`Manual_crear_nuevo_modulo_CRUD.md`** — *Manual: crear un nuevo módulo CRUD en APP API*. Aporta el procedimiento concreto del repositorio de referencia: arquitectura hexagonal con FastAPI, Pydantic v2 y SQLAlchemy; pasos de entidad/puerto/errores, modelo/mapper/repositorio, migraciones Alembic, DTO/casos de uso, providers, router, registro, pruebas y verificación; además de variantes de autenticación, FK simple/batch, N:M, jerarquías, borrado con referentes y matrices de contrato HTTP.

### 20.2 Consolidación de conceptos coincidentes

| Tema | Consolidación aplicada |
|---|---|
| Modelo de datos | Analizar esquema real, PK/FK, cardinalidad, unicidad, nulabilidad y política de eliminación antes de exponer endpoints. |
| Contrato REST | Definir método/ruta/DTO/status por endpoint y conservar coherencia del proyecto. |
| Seguridad | Autenticación separada de autorización; permiso de operación separado de propiedad/ámbito. |
| Validación | Schema de entrada, reglas de aplicación/dominio e integridad de BD tienen responsabilidades distintas. |
| Arquitectura | Router delgado, casos de uso, puertos, repositorios, mappers y handlers de error. |
| Listados | Límites, filtros consistentes entre `list` y `count`, orden estable y consultas seguras. |
| Errores | Errores tipados y mapeo centralizado al contrato HTTP. |
| Calidad | Pruebas positivas/negativas, seguridad, documentación y checklist de entrega. |

### 20.3 Recomendaciones adicionales incorporadas

Las siguientes recomendaciones amplían los documentos para cumplir el alcance genérico solicitado; no deben interpretarse como requisitos explícitos de ambos archivos originales:

- Matriz más amplia de métodos HTTP y decisión explícita de idempotencia.
- Consideraciones de `ETag`/`If-Match`, claves de idempotencia y versionado cuando sean pertinentes.
- Clasificación ampliada de validaciones sintácticas, estructurales, semánticas, de autorización y transición de estado.
- Advertencia explícita sobre las carreras de `MAX(id)+1` y la necesidad de proteger la unicidad bajo concurrencia.
- Ejemplos genéricos de `CHECK`, control de versión, Transactional Outbox y checklist con estado por fila.
- Referencia a OWASP API Security como marco adicional para riesgos de API.

### 20.4 Particularidades del segundo documento que no son reglas universales

El manual de `APP API` describe decisiones propias del repositorio y no deben trasladarse sin verificación a otros proyectos:

- Capas y rutas concretas bajo `src/domain`, `src/application`, `src/infrastructure` y `src/interfaces`.
- FastAPI, SQLAlchemy, Pydantic v2, Alembic, archivos providers y registro de router.
- Uso de `require_permissions`, `require_user_permissions`, `get_current_user`, permisos legacy y posibles bypass de administrador.
- Convención de listado `{items,total,offset,limit}` para algunos módulos y variantes `{data,total}` en otros.
- `PATCH` como actualización existente, `DELETE` con `204`, validación Pydantic con `422` y errores de dominio/FK a menudo `400`.
- Tablas `SI_*`/`USERS_*`, `LEGACY_TABLES`, migraciones idempotentes/no-op justificadas y algunos IDs asignados manualmente.
- Borrado físico como patrón actual, sin rate limiting o versionado `/v1` implementados en ese repositorio.

### 20.5 Diferencias y puntos que requieren decisión

1. **FK inválida y códigos HTTP:** el primer documento diferencia recurso inexistente y error de negocio de forma conceptual; el segundo mapea una FK inválida en el input a `400`, mientras que algunos errores `*NotFoundError` son `404`. No hay un único contrato universal. Esta guía recomienda documentar la diferencia: recurso solicitado inexistente suele ser `404`; referencia inválida en entrada puede ser `400` o `422`, según el estándar de la API. En el proyecto existente debe mantenerse el mapeo real hasta que se acuerde un cambio compatible.
2. **Permisos por acción frente a permiso de módulo:** el primer documento muestra permisos como `comments:create/read/update/delete`; el segundo también describe guards legacy basados en un slug general de módulo, como `estados` o `roles`. Son modelos de granularidad diferentes. No deben mezclarse sin definir si el guard de módulo se complementa con autorización por operación y por recurso.
3. **Formato de listado:** el primer documento utiliza ejemplos con `page/page_size` y también offset/cursor; el segundo describe `offset/limit`, además de envoltorios distintos por router. La guía no impone uno solo: debe elegirse según el contrato existente y las necesidades de volumen/UX.
4. **IDs legacy:** el segundo documento describe `MAX(ID)+1` por limitaciones de tablas existentes. Es una adaptación del legado, no una estrategia recomendada para nuevos diseños sin sincronización; puede producir colisiones concurrentes.
5. **Migraciones y borrado:** las migraciones no-op, exclusión de tablas en `LEGACY_TABLES` y la ausencia de soft delete son condiciones del repositorio de referencia. Cada proyecto debe revisar quién administra el esquema y qué política de retención necesita.

### 20.6 Vacíos no resolubles solo con los documentos

Los archivos no definen reglas universales para límites de paginación, tamaño máximo de contenido, estrategia de versionado, formato único de error, requisitos legales de auditoría, retención de datos, SLA, políticas concretas por organización ni modelo de despliegue. Estos puntos deben definirse por proyecto. Los valores usados en ejemplos de esta guía son ilustrativos.

### 20.7 Referencia externa

- [OWASP API Security](https://owasp.org/API-Security/) — referencia adicional de seguridad incorporada para complementar los documentos fuente.

---

**Principio final:** un módulo está listo cuando sus contratos son claros, sus reglas se cumplen en el backend, sus recursos están protegidos por operación y ámbito, sus datos conservan integridad, los errores son predecibles y las pruebas demuestran tanto el comportamiento esperado como el rechazo de operaciones no autorizadas.
