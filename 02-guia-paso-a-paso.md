# 02 · Guía paso a paso: implementar un módulo

Guía general para implementar **cualquier módulo** nuevo siguiendo la
arquitectura de una API. Cada paso indica: qué crear, dónde, por qué y un
snippet de código basado en el patrón real de los módulos `users`/`auth`.

> El ejemplo canónico de este documento es el módulo `users`. El recorrido
> completo con un módulo ficticio `tasks` está en
> [03-ejemplo-completo-tasks.md](03-ejemplo-completo-tasks.md).

## Resumen de pasos

| Paso | Capa            | Archivos                                                                        |
| ---- | --------------- | ------------------------------------------------------------------------------- |
| 0    | —               | Diseño previo (sin código)                                                      |
| 1    | Dominio         | `domain/entities/`, `domain/enums/`, `domain/errors.py`, `domain/ports/`        |
| 2    | Aplicación      | `application/dto/`, `application/use_cases/`                                    |
| 3    | Infraestructura | `database/models.py`, `database/mappers.py`, `repositories/`, migración Alembic |
| 4    | Interfaces      | `dependencies/providers.py`, `routers/`, `app.py`, `exception_handlers.py`      |
| 5    | Tests           | `tests/unit/`, `tests/integration/`                                             |
| 6    | Verificación    | `pytest` · `ruff` · `mypy`                                                      |
| 7    | Documentación   | `README.md` (tabla de endpoints)                                                |

**Orden estricto**: 1 → 2 → 3 → 4. Cada capa solo puede depender de las
anteriores (ver [01-arquitectura.md](01-arquitectura.md)).

---

## Paso 0 · Diseño previo

Antes de escribir código, responde:

1. **Entidad**: ¿qué cosa persiste? Ej.: `Task(title, done, user_id)`.
2. **Atributos**: tipos, restricciones (longitudes, unicidad), ¿cuáles son
   sensibles (passwords) y no deben salir en respuestas?
3. **Operaciones**: lista de casos de uso (create, get, list, update, delete…).
4. **Reglas de negocio**: validaciones que van al dominio/aplicación
   (p. ej. "no puedes borrarte a ti mismo").
5. **Autorización**: ¿quién puede cada operación? (público, autenticado, rol).
6. **Relaciones**: ¿FK a otras tablas? ¿se indexa?
7. **Errores esperados**: not-found, already-exists, conflicto…

Con eso decides el contenido de cada paso. Si no hay nada nuevo que diseñar
(módulo CRUD puro), sigue el patrón `users` tal cual.

---

## Paso 1 · Capa de dominio (`src/domain/`)

El dominio es Python puro: **cero imports** de SQLAlchemy, FastAPI, Pydantic o
cualquier librería externa.

### 1.1 Entidad — `src/domain/entities/<entidad>.py`

Dataclass con `slots=True`, `id: UUID` autogenerado, timestamps con
`default_factory` en UTC. Los campos obligatorios van primero; los opcionales
con default después.

```python
# src/domain/entities/user.py
from dataclasses import dataclass, field
from datetime import UTC, datetime
from uuid import UUID, uuid4

from src.domain.enums.role import Role


@dataclass(slots=True)
class User:
    username: str
    password_hash: str
    role: Role
    id: UUID = field(default_factory=uuid4)
    is_active: bool = True
    created_at: datetime = field(default_factory=lambda: datetime.now(UTC))
    updated_at: datetime = field(default_factory=lambda: datetime.now(UTC))
```

Detalles importantes:

- `id` usa `uuid4` por defecto; **no** se genera en la capa de persistencia.
- `created_at`/`updated_at` se actualizan con `datetime.now(UTC)` (aware).
- Si hay un campo sensible (`password_hash`), el DTO de respuesta **no lo
  incluye** (se expone en Paso 2).

### 1.2 Enum (si aplica) — `src/domain/enums/<nombre>.py`

`StrEnum` con un `parse()` que lanza `ValueError` con mensaje claro (lo usan
los mappers al leer de la BD):

```python
# src/domain/enums/role.py
from enum import StrEnum


class Role(StrEnum):
    ADMIN = "admin"
    USER = "user"

    @classmethod
    def parse(cls, value: str) -> Role:
        try:
            return cls(value)
        except ValueError:
            raise ValueError(f"unknown role: {value!r}") from None
```

### 1.3 Errores de dominio — `src/domain/errors.py`

Añade tus errores **heredando de `DomainError`** (o de un error intermedio si
el grupo ya existe, p. ej. `TokenError`):

```python
# src/domain/errors.py (fragmento: añade los tuyos)
class DomainError(Exception):
    """Base class for all domain errors."""


class UserNotFoundError(DomainError):
    def __init__(self, identifier: str) -> None:
        super().__init__(f"user not found: {identifier}")


class UserAlreadyExistsError(DomainError):
    def __init__(self, username: str) -> None:
        super().__init__(f"user already exists: {username}")
```

Cada error lleva un constructor que arma el mensaje (no uses `raise DomainError("...")`
con strings sueltos fuera de esta capa). Su mapeo a HTTP se registra en el Paso 4.

### 1.4 Puerto repositorio — `src/domain/ports/<entidad>_repository.py`

Interfaz `ABC` con `@abstractmethod` por cada operación que la aplicación
necesita. Devuelve **entidades de dominio**, nunca modelos ORM:

```python
# src/domain/ports/user_repository.py
from abc import ABC, abstractmethod
from uuid import UUID

from src.domain.entities.user import User
from src.domain.enums.role import Role


class UserRepository(ABC):
    @abstractmethod
    def create(self, user: User) -> None: ...

    @abstractmethod
    def get_by_username(self, username: str) -> User | None: ...

    @abstractmethod
    def get_by_id(self, user_id: UUID) -> User | None: ...

    @abstractmethod
    def list(self, offset: int = 0, limit: int = 100) -> list[User]: ...

    @abstractmethod
    def count(self) -> int: ...

    @abstractmethod
    def set_role(self, user_id: UUID, role: Role) -> User: ...

    @abstractmethod
    def delete(self, user_id: UUID) -> None: ...
```

Convenciones:

- `create` devuelve `None` (la entidad ya se modificó en memoria si hace falta).
- Las lecturas devuelven `X | None`; **el repositorio no lanza "not found"** en
  getters — quien decide si es 404 es el caso de uso. Excepción: operaciones de
  escritura directa sobre un ID (`set_role`, `delete`) sí lanzan
  `XNotFoundError` porque no hay un paso previo de lectura (ver
  `sqlalchemy_user_repository.py`).
- `list` siempre lleva `offset`/`limit` y un `count()` separado para el total.

### 1.5 Otros puertos (si aplica)

Si el módulo necesita un servicio tecnológico (hash, cripto, HTTP externo…),
define el puerto en `src/domain/ports/` y su adaptador en infraestructura:

```python
# src/domain/ports/password_hasher.py
from abc import ABC, abstractmethod


class PasswordHasher(ABC):
    @abstractmethod
    def hash(self, password: str) -> str: ...

    @abstractmethod
    def verify(self, password: str, password_hash: str) -> bool: ...
```

---

## Paso 2 · Capa de aplicación (`src/application/`)

### 2.1 DTOs — `src/application/dto/<entidad>_dto.py`

Modelos Pydantic v2 que definen el **contrato HTTP** (lo que entra y lo que
sale). Nunca expongan campos sensibles.

```python
# src/application/dto/user_dto.py
from datetime import datetime
from uuid import UUID

from pydantic import BaseModel, Field

from src.domain.entities.user import User
from src.domain.enums.role import Role

_USERNAME_PATTERN = r"^[a-zA-Z0-9_.-]+$"


class UserCreate(BaseModel):
    username: str = Field(min_length=3, max_length=50, pattern=_USERNAME_PATTERN)
    password: str = Field(min_length=8, max_length=128)
    role: Role = Role.USER


class RoleUpdateRequest(BaseModel):
    role: Role


class UserResponse(BaseModel):
    id: UUID
    username: str
    role: Role
    is_active: bool
    created_at: datetime

    @classmethod
    def from_entity(cls, user: User) -> UserResponse:
        return cls(
            id=user.id,
            username=user.username,
            role=user.role,
            is_active=user.is_active,
            created_at=user.created_at,
        )


class UserListResponse(BaseModel):
    items: list[UserResponse]
    total: int
    offset: int
    limit: int
```

Convenciones de DTO:

- **Request de creación**: `<Entidad>Create`; con `Field` para longitudes y
  patrones (Pydantic devuelve 422 automáticamente al fallar la validación).
- **Response**: `<Entidad>Response` con `from_entity(cls, entity)` para
  centralizar la conversión entidad → respuesta.
- **Lista**: `<Entidad>ListResponse` con `items`, `total`, `offset`, `limit`.
- El `password_hash` de `User` **no aparece** en `UserResponse`.

### 2.2 Casos de uso — `src/application/use_cases/<verbo>_<entidad>.py`

Una clase por caso de uso: recibe sus puertos por constructor y expone
`execute(...)`. Aquí vive la orquestación y las reglas de negocio.

**Patrón base (CRUD simple):**

```python
# src/application/use_cases/create_user.py
from src.application.dto.user_dto import UserCreate, UserResponse
from src.domain.entities.user import User
from src.domain.ports.password_hasher import PasswordHasher
from src.domain.ports.user_repository import UserRepository


class CreateUserUseCase:
    def __init__(
        self,
        user_repository: UserRepository,
        password_hasher: PasswordHasher,
    ) -> None:
        self._users = user_repository
        self._hasher = password_hasher

    def execute(self, dto: UserCreate) -> UserResponse:
        user = User(
            username=dto.username,
            password_hash=self._hasher.hash(dto.password),
            role=dto.role,
        )
        self._users.create(user)
        return UserResponse.from_entity(user)
```

**Patrón de lista con paginación acotada:**

```python
# src/application/use_cases/list_users.py
from src.application.dto.user_dto import UserListResponse, UserResponse
from src.domain.ports.user_repository import UserRepository


class ListUsersUseCase:
    def __init__(self, user_repository: UserRepository) -> None:
        self._users = user_repository

    def execute(self, offset: int = 0, limit: int = 100) -> UserListResponse:
        limit = max(1, min(limit, 200))
        offset = max(0, offset)
        users = self._users.list(offset=offset, limit=limit)
        return UserListResponse(
            items=[UserResponse.from_entity(user) for user in users],
            total=self._users.count(),
            offset=offset,
            limit=limit,
        )
```

**Patrón multi-repositorio (efectos colaterales):**

```python
# src/application/use_cases/delete_user.py
from uuid import UUID

from src.domain.ports.token_repository import TokenRepository
from src.domain.ports.user_repository import UserRepository


class DeleteUserUseCase:
    def __init__(
        self,
        user_repository: UserRepository,
        token_repository: TokenRepository,
    ) -> None:
        self._users = user_repository
        self._tokens = token_repository

    def execute(self, user_id: UUID) -> None:
        self._users.delete(user_id)
        self._tokens.revoke_all_for_user(user_id)
```

**Reglas para casos de uso:**

- Importan de `application/dto` y `domain/*` **nunca** de `infrastructure` ni
  `interfaces`.
- Si necesitan saber si algo existe, hacen la lectura y lanzan el error de
  dominio correspondiente (`raise UserNotFoundError(str(user_id))`).
- El DTO de entrada se valida en FastAPI (Paso 4); el caso de uso puede
  confiar en él, pero las reglas de negocio van aquí.
- Los parámetros escalares van como argumentos de `execute` cuando son pocos
  (`user_id: UUID, role: Role`); un DTO completo cuando la request es rica.

---

## Paso 3 · Capa de infraestructura (`src/infrastructure/`)

### 3.1 Modelo ORM — `src/infrastructure/database/models.py`

Añade tu modelo a este archivo compartido (no crees archivos de modelos
nuevos: `alembic/env.py` importa `models` para el autogenerate).

```python
# src/infrastructure/database/models.py (fragmento)
from datetime import UTC, datetime

from sqlalchemy import Boolean, CheckConstraint, DateTime, ForeignKey, String
from sqlalchemy.orm import Mapped, mapped_column

from src.infrastructure.database.base import Base


def utcnow() -> datetime:
    return datetime.now(UTC)


class UserModel(Base):
    __tablename__ = "users"
    __table_args__ = (
        CheckConstraint("role IN ('admin', 'user')", name="ck_users_role"),
    )

    id: Mapped[str] = mapped_column(String(36), primary_key=True)
    username: Mapped[str] = mapped_column(String(50), unique=True, index=True)
    password_hash: Mapped[str] = mapped_column(String(128))
    role: Mapped[str] = mapped_column(String(20))
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), default=utcnow, onupdate=utcnow
    )
```

Reglas:

- PK: `String(36)` con el `UUID` serializado a `str`.
- FK: `String(36)` + `ForeignKey("tabla.id", ondelete="CASCADE")` + `index=True`.
  El `ondelete` debe alinearse con el comportamiento deseado (SQLite necesita
  `PRAGMA foreign_keys=ON`, que ya activa `session.py`).
- Enums guardados como `String` + `CheckConstraint(..., name="ck_<tabla>_<col>")`.
- Defaults de fecha con la función `utcnow` del propio archivo;
  `onupdate=utcnow` para `updated_at`.
- `unique=True, index=True` en campos con unicidad de negocio.

### 3.2 Mappers — `src/infrastructure/database/mappers.py`

Funciones puras `x_to_entity(model)` y `x_to_model(entity)`. Úsalas
exclusivamente dentro de los repositorios:

```python
# src/infrastructure/database/mappers.py (fragmento)
from datetime import UTC, datetime
from uuid import UUID

from src.domain.entities.user import User
from src.domain.enums.role import Role
from src.infrastructure.database.models import UserModel


def as_utc(value: datetime) -> datetime:
    """SQLite drops tzinfo on storage; normalize to aware UTC."""
    if value.tzinfo is None:
        return value.replace(tzinfo=UTC)
    return value.astimezone(UTC)


def user_to_entity(model: UserModel) -> User:
    return User(
        id=UUID(model.id),
        username=model.username,
        password_hash=model.password_hash,
        role=Role.parse(model.role),
        is_active=model.is_active,
        created_at=as_utc(model.created_at),
        updated_at=as_utc(model.updated_at),
    )


def user_to_model(entity: User) -> UserModel:
    return UserModel(
        id=str(entity.id),
        username=entity.username,
        password_hash=entity.password_hash,
        role=entity.role.value,
        is_active=entity.is_active,
        created_at=entity.created_at,
        updated_at=entity.updated_at,
    )
```

Detalles:

- `UUID(...)` / `str(...)` en los IDs; `.value` / `Enum.parse()` en los enums.
- **Siempre** envuelve fechas leídas con `as_utc()` (SQLite descarta `tzinfo`).

### 3.3 Repositorio — `src/infrastructure/repositories/sqlalchemy_<entidad>_repository.py`

Hereda de `SqlAlchemyRepository` (da el context manager `_session`) y del
puerto de dominio:

```python
# src/infrastructure/repositories/sqlalchemy_user_repository.py (fragmento)
from uuid import UUID

from sqlalchemy import func, select

from src.domain.entities.user import User
from src.domain.enums.role import Role
from src.domain.errors import UserAlreadyExistsError, UserNotFoundError
from src.domain.ports.user_repository import UserRepository
from src.infrastructure.database.mappers import user_to_entity, user_to_model
from src.infrastructure.database.models import UserModel
from src.infrastructure.repositories.base import SqlAlchemyRepository


class SqlAlchemyUserRepository(SqlAlchemyRepository, UserRepository):
    def create(self, user: User) -> None:
        with self._session(commit=True) as session:
            exists = session.scalar(
                select(UserModel.id).where(UserModel.username == user.username)
            )
            if exists:
                raise UserAlreadyExistsError(user.username)
            session.add(user_to_model(user))

    def get_by_id(self, user_id: UUID) -> User | None:
        with self._session() as session:
            model = session.get(UserModel, str(user_id))
            return user_to_entity(model) if model else None

    def list(self, offset: int = 0, limit: int = 100) -> list[User]:
        with self._session() as session:
            models = session.scalars(
                select(UserModel)
                .order_by(UserModel.created_at.desc(), UserModel.username.asc())
                .offset(offset)
                .limit(limit)
            ).all()
            return [user_to_entity(model) for model in models]

    def count(self) -> int:
        with self._session() as session:
            return int(session.scalar(select(func.count()).select_from(UserModel)) or 0)

    def set_role(self, user_id: UUID, role: Role) -> User:
        with self._session(commit=True) as session:
            model = session.get(UserModel, str(user_id))
            if not model:
                raise UserNotFoundError(str(user_id))
            model.role = role.value
            session.flush()
            return user_to_entity(model)

    def delete(self, user_id: UUID) -> None:
        with self._session(commit=True) as session:
            model = session.get(UserModel, str(user_id))
            if not model:
                raise UserNotFoundError(str(user_id))
            session.delete(model)
```

Reglas:

- **Lecturas**: `with self._session()` (sin commit).
- **Escrituras**: `with self._session(commit=True)`. Ante excepción el context
  manager hace `rollback` y relanza (ver `repositories/base.py`).
- Validación de unicidad **antes** de `add` → `raise XAlreadyExistsError`.
- Escritura por ID inexistente → `raise XNotFoundError`.
- Mapea siempre con `*_to_entity` antes de devolver: el puerto promete entidades.
- Sin `session.flush()` explícito salvo que necesites el estado inmediato.

### 3.4 Migración Alembic

```bash
.venv/bin/alembic revision --autogenerate -m "add tasks table"
```

1. Alembic compara `Base.metadata` (los modelos importados en
   `alembic/env.py` → `from src.infrastructure.database import models`) contra
   la BD actual y genera `alembic/versions/<hash>_<mensaje>.py`.
2. **Revisa el diff generado** (autogenerate no detecta todo: `ondelete`,
   índices parciales, comentarios…). Ajusta `upgrade()` y escribe `downgrade()`.
3. Aplica:

```bash
.venv/bin/alembic upgrade head
```

En desarrollo no es obligatorio ejecutarlo a mano: las migraciones corren en el
lifespan de la app (`run_migrations()` en `src/interfaces/api/app.py`).

Ejemplo de migración real: `alembic/versions/9a6d2620ab36_create_users_and_refresh_tokens.py`.

---

## Paso 4 · Capa de interfaces (`src/interfaces/api/`)

### 4.1 Providers (composition root) — `dependencies/providers.py`

Aquí se decide qué implementación concreta recibe cada caso de uso. Añade una
función `get_<caso>_use_case()` por caso nuevo:

```python
# src/interfaces/api/dependencies/providers.py (fragmento)
def get_user_repository() -> UserRepository:
    return SqlAlchemyUserRepository(get_database())


def get_create_user_use_case() -> CreateUserUseCase:
    return CreateUserUseCase(
        user_repository=get_user_repository(),
        password_hasher=get_password_hasher(),
    )
```

Convenciones:

- Repositorios y casos de uso: función normal (se instancian por request).
- Objetos caros/cíclicos (`get_database`, `get_password_hasher`,
  `get_token_provider`): `@lru_cache(maxsize=1)`.
- La configuración llega vía `get_settings()`; los TTL u otros números van como
  **argumentos escalares** al caso de uso (no un objeto `Settings` entero),
  para que el caso de uso siga siendo testeable sin pydantic-settings.

### 4.2 Router — `src/interfaces/api/routers/<entidad>_router.py`

```python
# src/interfaces/api/routers/users_router.py (fragmento)
from uuid import UUID

from fastapi import APIRouter, Depends, HTTPException

from src.application.dto.user_dto import (
    RoleUpdateRequest,
    UserCreate,
    UserListResponse,
    UserResponse,
)
from src.application.use_cases.create_user import CreateUserUseCase
from src.application.use_cases.delete_user import DeleteUserUseCase
from src.application.use_cases.list_users import ListUsersUseCase
from src.application.use_cases.update_user_role import UpdateUserRoleUseCase
from src.domain.entities.user import User
from src.domain.enums.role import Role
from src.interfaces.api.dependencies.auth import require_roles
from src.interfaces.api.dependencies.providers import (
    get_create_user_use_case,
    get_delete_user_use_case,
    get_list_users_use_case,
    get_update_user_role_use_case,
)

router = APIRouter(prefix="/users", tags=["users"])

admin_only = require_roles(Role.ADMIN)


@router.get("", response_model=UserListResponse)
def list_users(
    offset: int = 0,
    limit: int = 100,
    _: User = Depends(admin_only),
    use_case: ListUsersUseCase = Depends(get_list_users_use_case),
) -> UserListResponse:
    return use_case.execute(offset=offset, limit=limit)


@router.post("", response_model=UserResponse, status_code=201)
def create_user(
    dto: UserCreate,
    _: User = Depends(admin_only),
    use_case: CreateUserUseCase = Depends(get_create_user_use_case),
) -> UserResponse:
    return use_case.execute(dto)


@router.delete("/{user_id}", status_code=204)
def delete_user(
    user_id: UUID,
    actor: User = Depends(admin_only),
    use_case: DeleteUserUseCase = Depends(get_delete_user_use_case),
) -> None:
    use_case.execute(user_id)
```

Reglas del router:

- `APIRouter(prefix="/<recurso>", tags=["<recurso>"])`.
- `response_model` siempre explícito; `status_code=201` en creación, `204` en
  DELETE (sin cuerpo → retorno `None`).
- Path params tipados (`user_id: UUID`) → FastAPI devuelve 422 si no es UUID.
- Query params con default (`offset: int = 0, limit: int = 100`).
- El router solo llama `use_case.execute(...)`; cualquier `HTTPException` aquí
  es de **transporte** (reglas de actor, p. ej. no borrarte a ti mismo), no de
  negocio:

```python
def _guard_self_action(actor: User, target: UUID, action: str) -> None:
    if actor.id == target:
        raise HTTPException(status_code=400, detail=f"Cannot {action} yourself")
```

### 4.3 Autorización — `dependencies/auth.py`

Reutiliza los guards existentes; no reimplementes JWT:

```python
# Público
@router.get("/algo", ...)
def endpoint(...): ...

# Autenticado (cualquier usuario activo)
def endpoint(user: User = Depends(get_current_user), ...): ...

# Por rol
admin_only = require_roles(Role.ADMIN)

def endpoint(_: User = Depends(admin_only), ...): ...
```

- `get_current_user` decodifica el access token y carga el usuario (401 si
  falta/expira/inactivo).
- `require_roles(*roles)` envuelve `get_current_user` y devuelve 403 si el rol
  no coincide. Se puede combinar: `require_roles(Role.ADMIN, Role.USER)`.

### 4.4 Manejo de errores nuevos — `exception_handlers.py`

Si añadiste errores de dominio en el Paso 1.3, regístralos con su status:

```python
# src/interfaces/api/exception_handlers.py (añade DENTRO de register_exception_handlers)
@app.exception_handler(TaskNotFoundError)
def _task_not_found(request: Request, exc: TaskNotFoundError) -> JSONResponse:
    return _error_response(404, str(exc))
```

Importa el error arriba del archivo junto a los demás. El handler base
`DomainError → 400` ya existe como fallback, pero cada error importante debe
tener su status correcto.

### 4.5 Registro del router — `src/interfaces/api/app.py`

```python
# src/interfaces/api/app.py (fragmento)
from src.interfaces.api.routers import auth_router, tasks_router, users_router
# ...
app.include_router(auth_router.router)
app.include_router(users_router.router)
app.include_router(tasks_router.router)
```

Sin este `include_router`, las rutas no existen. (`tasks_router` es ilustrativo;
añade solo los routers que crees.)

---

## Paso 5 · Tests

### 5.1 Unit — `tests/unit/test_<tema>.py`

Caso de uso + repositorios **reales** sobre SQLite de test (no mocks). Crea tu
propia fixture siguiendo el patrón `env` de `tests/unit/test_use_cases.py`:

```python
# tests/unit/test_use_cases.py (fragmento de la fixture)
from types import SimpleNamespace

import pytest

from src.infrastructure.config import Settings
from src.infrastructure.database.session import Database
from src.infrastructure.repositories.sqlalchemy_user_repository import (
    SqlAlchemyUserRepository,
)
from src.infrastructure.security.bcrypt_password_hasher import BcryptPasswordHasher


@pytest.fixture
def env():
    settings = Settings(_env_file=None, secret_key="unit-secret")
    database = Database(settings)
    database.create_all()
    users = SqlAlchemyUserRepository(database)
    hasher = BcryptPasswordHasher(rounds=4)
    return SimpleNamespace(
        database=database,
        users=users,
        hasher=hasher,
        create=CreateUserUseCase(users, hasher),
        list_users=ListUsersUseCase(users),
    )
```

Casos mínimos por operación: happy path, camino de error (no found /
already-exists) y validación de reglas de negocio:

```python
def test_create_user_persists_and_hides_hash(env):
    response = env.create.execute(UserCreate(username="alice", password="password1"))
    stored = env.users.get_by_id(response.id)
    assert stored.username == "alice"
    assert stored.password_hash != "password1"


def test_create_duplicate_user_raises(env):
    env.create.execute(UserCreate(username="alice", password="password1"))
    with pytest.raises(UserAlreadyExistsError):
        env.create.execute(UserCreate(username="alice", password="password1"))
```

### 5.2 Integración — `tests/integration/test_<área>_api.py`

Contra la app real usando las fixtures de `tests/conftest.py`
(`client`, `admin_headers`, `create_user`, `login`; la BD se resetea en cada
test con `db_reset`):

```python
# tests/integration/test_rbac_api.py (fragmento)
def test_unauthenticated_cannot_list_users(client):
    assert client.get("/users").status_code == 401


def test_regular_user_cannot_list_users(client, create_user, login):
    create_user("alice")
    headers, _ = login("alice", "password1")
    assert client.get("/users", headers=headers).status_code == 403


def test_create_duplicate_user_returns_409(client, admin_headers, create_user):
    create_user("alice")
    response = client.post(
        "/users",
        headers=admin_headers,
        json={"username": "alice", "password": "password1", "role": "user"},
    )
    assert response.status_code == 409
```

Cobertura mínima por endpoint nuevo:

1. **Happy path** con rol adecuado (200/201/204 + shape del body).
2. **Sin autenticación** → 401.
3. **Rol insuficiente** → 403.
4. **Recurso inexistente** → 404.
5. **Validación de entrada** → 422.
6. **Conflicto/duplicado** (si aplica) → 409.

---

## Paso 6 · Verificación

```bash
.venv/bin/pytest                    # todos los tests
.venv/bin/ruff check src tests main.py
.venv/bin/mypy src
```

Los tres deben pasar antes de continuar. Ruff ordena imports
(`known-first-party = ["src"]`) y respeta line-length 100.

---

## Paso 7 · Documentación del módulo

1. Añade cada endpoint nuevo a la tabla **Endpoints** de
   [`README.md`](../README.md):

   ```markdown
   | GET | `/tasks` | Bearer |
   | POST | `/tasks` | Bearer |
   ```

2. Si el módulo añade variables de entorno, actualiza `.env.example` y la
   sección _Environment Variables_ de `AGENTS.md`.
3. Si cambia el flujo de setup/tests, actualiza `AGENTS.md` (_Commands_,
   _Gotchas_).

---

## Errores comunes

| Síntoma                                           | Causa / arreglo                                                                                            |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| 404 en la ruta nueva                              | Falta `app.include_router(...)` en `app.py`, o el prefijo del `APIRouter` no coincide.                     |
| `alembic autogenerate` no ve tu tabla             | El modelo no está en `src/infrastructure/database/models.py` (es el único importado por `alembic/env.py`). |
| `sqlite3.OperationalError: no such table`         | No se generó/aplicó la migración; revisa `alembic upgrade head` y que los tests usen `create_all`.         |
| AttributeError de fecha sin timezone              | Falta `as_utc()` en el mapper al leer de la BD.                                                            |
| Error de dominio devuelve 400 y esperabas 404/409 | Falta registrar el handler en `exception_handlers.py`.                                                     |
| `ImportError` desde `domain/` o `application/`    | Estás importando infraestructura/interfaces desde una capa inferior: mueve la dependencia a un puerto.     |
| El endpoint ignora el rol                         | No añadiste `Depends(require_roles(...))` o `Depends(get_current_user)`.                                   |
| Test integración ve datos de otro test            | Debe existir la fixture `db_reset` (autouse en `conftest.py`); no la elimines.                             |

Siguiente: [03-ejemplo-completo-tasks.md](03-ejemplo-completo-tasks.md).
