# 01 · Arquitectura

API sigue **arquitectura hexagonal** (puertos y adaptadores) organizada en
cuatro capas. Este documento explica qué vive en cada una, cómo se conectan y las
convenciones que aplican a todo módulo.

## 1. Las cuatro capas

```
src/
├── domain/           # Reglas de negocio puras (sin dependencias externas)
│   ├── entities/     #   Entidades (dataclasses)
│   ├── enums/        #   Enumeraciones (StrEnum)
│   ├── ports/        #   Interfaces abstractas (ABC) = "puertos"
│   └── errors.py     #   Excepciones de dominio
├── application/      # Casos de uso + DTOs (orquesta el dominio)
│   ├── use_cases/    #   Una clase por caso de uso, con execute()
│   └── dto/          #   Modelos Pydantic de entrada/salida
├── infrastructure/   # Adaptadores concretos (BD, seguridad, config)
│   ├── database/     #   Modelos SQLAlchemy, mappers, sesión, migraciones
│   ├── repositories/ #   Implementación de los puertos de dominio
│   ├── security/     #   Hashers y proveedores de token
│   └── config.py     #   Settings (pydantic-settings)
└── interfaces/       # Transporte (FastAPI): routers, DI, handlers
    └── api/
        ├── routers/          # Endpoints HTTP
        ├── dependencies/     # Providers (composition root) + auth
        ├── exception_handlers.py
        ├── middleware.py
        └── app.py            # create_app(): ensambla todo
```

### Regla de dependencias

Las flechas indican **quién puede importar a quién**:

```
interfaces ──────────► application ──────────► domain ◄────────── infrastructure
     │                                            ▲                    │
     └────────────────────────────────────────────┘                    │
                    (interfaces también puede usar                     │
                     infrastructure vía providers)  ◄──────────────────┘
```

- **`domain` no importa nada** de las demás capas: es Python puro
  (`dataclasses`, `abc`, `enum`, `uuid`, `datetime`).
- **`application` importa solo `domain`** (entidades, puertos, DTOs propios).
- **`infrastructure` implementa los puertos de `domain`** (p. ej.
  `SqlAlchemyUserRepository` implementa `UserRepository`).
- **`interfaces` orquesta todo** en el _composition root_
  (`dependencies/providers.py`): es el único punto donde se decide qué
  implementación concreta recibe cada caso de uso.

En la práctica: **el dominio define el "qué" (puertos), la infraestructura el
"cómo" (adaptadores), la aplicación el "cuándo" (casos de uso) y las interfaces
el "desde dónde" (HTTP)**.

## 2. Flujo de una request

Ejemplo: `POST /users` (crear usuario).

```
HTTP POST /users
  │
  ▼
users_router.create_user()                interfaces/api/routers/users_router.py
  │  Depends(admin_only)                  → valida JWT + rol (dependencies/auth.py)
  │  Depends(get_create_user_use_case)    → construye el caso de uso (providers.py)
  ▼
CreateUserUseCase.execute(dto)            application/use_cases/create_user.py
  │  password_hasher.hash(dto.password)   → puerto PasswordHasher
  │  user_repository.create(user)         → puerto UserRepository
  ▼
SqlAlchemyUserRepository.create(user)     infrastructure/repositories/...
  │  user_to_model(user)                  → mapper (infrastructure/database/mappers.py)
  │  session.add(...) / commit            → SQLAlchemy
  ▼
DTO UserResponse.from_entity(user)        ← vuelve por application hasta el router
  │
  ▼
201 Created  {"id": ..., "username": ...}
```

Puntos clave del flujo:

1. El router **no contiene lógica de negocio**: solo valida transporte
   (auth, status code, parámetros) y delega en `use_case.execute(...)`.
2. El caso de uso **solo conoce puertos** (`UserRepository`, `PasswordHasher`),
   nunca clases de SQLAlchemy.
3. La conversión entidad ⇄ modelo la hacen los **mappers** de infraestructura;
   la conversión entidad ⇄ JSON la hacen los **DTOs** de aplicación.

## 3. Manejo de errores

El dominio lanza excepciones propias (heredadas de `DomainError` en
`src/domain/errors.py`) y FastAPI las traduce a HTTP en
`src/interfaces/api/exception_handlers.py`:

| Excepción de dominio                                      | HTTP | Ejemplo de uso                    |
| --------------------------------------------------------- | ---- | --------------------------------- |
| `InvalidCredentialsError`                                 | 401  | login con password incorrecto     |
| `TokenError` (→ `InvalidTokenError`, `TokenExpiredError`) | 401  | JWT inválido o expirado           |
| `ForbiddenError`                                          | 403  | rol insuficiente                  |
| `UserNotFoundError`                                       | 404  | recurso inexistente               |
| `UserAlreadyExistsError`                                  | 409  | username duplicado                |
| `DomainError` (base)                                      | 400  | fallback para errores no mapeados |

Reglas:

- Los **casos de uso y repositorios lanzan errores de dominio**, nunca
  `HTTPException`.
- `HTTPException` **solo aparece en `interfaces/`** (routers y dependencies),
  para violaciones de contrato HTTP (401/403/400 de validación de transporte).
- Al añadir un error de dominio nuevo, regístralo en
  `exception_handlers.py` con su status; si no, caerá al handler base
  `DomainError` → 400.

## 4. Convenciones globales

### Naming

| Elemento             | Convención                                         | Ejemplo real                      |
| -------------------- | -------------------------------------------------- | --------------------------------- |
| Entidad              | sustantivo, `PascalCase`                           | `User`, `RefreshToken`            |
| Puerto (repositorio) | sustantivo + `Repository`                          | `UserRepository`                  |
| Puerto (servicio)    | sustantivo + rol técnico                           | `PasswordHasher`, `TokenProvider` |
| Adaptador SQLAlchemy | `SqlAlchemy` + puerto                              | `SqlAlchemyUserRepository`        |
| Modelo ORM           | entidad + `Model`                                  | `UserModel`                       |
| Caso de uso          | verbo + sustantivo + `UseCase`, método `execute()` | `CreateUserUseCase.execute()`     |
| DTO request          | sustantivo + `Create`/`...Request`                 | `UserCreate`, `LoginRequest`      |
| DTO response         | sustantivo + `Response`                            | `UserResponse`, `TokenResponse`   |
| DTO de lista         | sustantivo + `ListResponse`                        | `UserListResponse`                |
| Provider (DI)        | `get_` + sustantivo + `_use_case` / `_repository`  | `get_create_user_use_case()`      |
| Archivo de router    | sustantivo + `_router`                             | `users_router.py`                 |
| Test unit            | `test_<tema>.py`                                   | `test_use_cases.py`               |
| Test integración     | `test_<área>_api.py`                               | `test_rbac_api.py`                |

### Tipos y valores

- **IDs**: `UUID` en dominio/aplicación; `String(36)` (`str(uuid)`) en la BD.
- **Fechas**: `datetime` con timezone UTC (`datetime.now(UTC)`); los mappers
  normalizan con `as_utc()` porque SQLite descarta `tzinfo`.
- **Enums**: `StrEnum` con un `.parse()` amigable que lanza `ValueError`
  (ver `src/domain/enums/role.py`). En la BD se guarda `.value: str` y, si hay
  restricción, un `CheckConstraint` en el modelo.
- **Endpoints**: funciones síncronas (`def`, no `async def`); todo el stack es
  síncrono (SQLAlchemy sync, repositorios sync).
- **Paginación**: los casos de uso hacen _clamp_ de límites
  (`limit = max(1, min(limit, 200))`); la respuesta de lista siempre incluye
  `items`, `total`, `offset`, `limit`.
- **Estilo**: line-length 100, ruff con `E,F,I,N,UP,B,SIM` (ver
  `pyproject.toml`), imports de primero `src` (`known-first-party`).

### Inyección de dependencias

- Los casos de uso reciben sus puertos por **constructor** (`__init__`).
- Los providers (`src/interfaces/api/dependencies/providers.py`) son funciones
  puras que componen el grafo; los objetos de ciclo de vida largo (database,
  hasher, token provider) usan `@lru_cache(maxsize=1)`.
- Los routers no instancian nada: usan `Depends(get_x_use_case)`.

### Tests

- **Unit** (`tests/unit/`): caso de uso + repositorios reales sobre SQLite de
  test (fixture `env` en `tests/unit/test_use_cases.py`).
- **Integración** (`tests/integration/`): `TestClient` contra la app real con
  fixtures de `tests/conftest.py` (`client`, `admin_headers`, `create_user`,
  `login`). La base se resetea en cada test (`db_reset`, autouse).

## 5. Puntos de extensión por tipo de módulo

| Si tu módulo…             | Toca además…                                                                                                     |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Es CRUD simple            | Solo los 4 pasos base (dominio, aplicación, infraestructura, interfaces)                                         |
| Tiene datos propios en BD | Migración Alembic (`alembic revision --autogenerate`)                                                            |
| Requiere autenticación    | Reutiliza `get_current_user` / `require_roles` (no reimplementes auth)                                           |
| Requiere roles            | `require_roles(Role.ADMIN)` en el router; nuevo rol → `src/domain/enums/role.py` + `CheckConstraint` + migración |
| Tiene nuevo tipo de error | `src/domain/errors.py` + handler en `exception_handlers.py`                                                      |
| Requiere config nueva     | Campo en `src/infrastructure/config.py` + `.env.example`                                                         |
| Expone endpoints          | `app.include_router(...)` en `src/interfaces/api/app.py` + fila en la tabla del `README.md`                      |

Siguiente: [02-guia-paso-a-paso.md](02-guia-paso-a-paso.md).
