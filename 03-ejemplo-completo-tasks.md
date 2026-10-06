# 03 · Ejemplo completo: módulo `tasks`

Recorrido end-to-end de un módulo ficticio siguiendo
[02-guia-paso-a-paso.md](02-guia-paso-a-paso.md). Es plantilla de copiar y
adaptar: **estos archivos no existen en `src/`** (solo es documentación).

## Diseño previo (Paso 0)

| Decisión | Respuesta |
|---|---|
| Entidad | `Task`: una tarea pertenece a un usuario |
| Atributos | `title` (1–100), `description` (0–500, opcional), `done` (bool), `user_id` (FK → users), timestamps |
| Operaciones | create, list (propias), mark_done, delete |
| Reglas | Solo ves/borras tus tareas; `title` no puede estar vacío |
| Autorización | Todo autenticado (`get_current_user`); sin roles especiales |
| Relaciones | `user_id` FK → `users.id`, `ondelete=CASCADE`, indexada |
| Errores | `TaskNotFoundError` (404). Propiedad ajena → también 404 (no filtrar existencia) |
| Sin enum nuevo | Se usa `done: bool`; el patrón `StrEnum` está en `role.py` |

---

## Paso 1 · Dominio

### 1.1 Entidad — `src/domain/entities/task.py`

```python
from dataclasses import dataclass, field
from datetime import UTC, datetime
from uuid import UUID, uuid4


@dataclass(slots=True)
class Task:
    title: str
    user_id: UUID
    id: UUID = field(default_factory=uuid4)
    description: str = ""
    done: bool = False
    created_at: datetime = field(default_factory=lambda: datetime.now(UTC))
    updated_at: datetime = field(default_factory=lambda: datetime.now(UTC))
```

### 1.2 Errores — `src/domain/errors.py` (añade al final)

```python
class TaskNotFoundError(DomainError):
    def __init__(self, identifier: str) -> None:
        super().__init__(f"task not found: {identifier}")
```

### 1.3 Puerto — `src/domain/ports/task_repository.py`

```python
from abc import ABC, abstractmethod
from uuid import UUID

from src.domain.entities.task import Task


class TaskRepository(ABC):
    @abstractmethod
    def create(self, task: Task) -> None: ...

    @abstractmethod
    def get_by_id(self, task_id: UUID) -> Task | None: ...

    @abstractmethod
    def list_for_user(
        self, user_id: UUID, offset: int = 0, limit: int = 100
    ) -> list[Task]: ...

    @abstractmethod
    def count_for_user(self, user_id: UUID) -> int: ...

    @abstractmethod
    def set_done(self, task_id: UUID, done: bool) -> Task: ...

    @abstractmethod
    def delete(self, task_id: UUID) -> None: ...
```

> Nota de diseño: `list_for_user`/`count_for_user` filtran por dueño en la BD
> (eficiente y seguro). `set_done`/`delete` trabajan por ID y la verificación de
> propiedad ocurre en el caso de uso.

---

## Paso 2 · Aplicación

### 2.1 DTOs — `src/application/dto/task_dto.py`

```python
from datetime import datetime
from uuid import UUID

from pydantic import BaseModel, Field

from src.domain.entities.task import Task


class TaskCreate(BaseModel):
    title: str = Field(min_length=1, max_length=100)
    description: str = Field(default="", max_length=500)


class TaskResponse(BaseModel):
    id: UUID
    title: str
    description: str
    done: bool
    user_id: UUID
    created_at: datetime
    updated_at: datetime

    @classmethod
    def from_entity(cls, task: Task) -> TaskResponse:
        return cls(
            id=task.id,
            title=task.title,
            description=task.description,
            done=task.done,
            user_id=task.user_id,
            created_at=task.created_at,
            updated_at=task.updated_at,
        )


class TaskListResponse(BaseModel):
    items: list[TaskResponse]
    total: int
    offset: int
    limit: int
```

### 2.2 Casos de uso

#### `src/application/use_cases/create_task.py`

```python
from uuid import UUID

from src.application.dto.task_dto import TaskCreate, TaskResponse
from src.domain.entities.task import Task
from src.domain.ports.task_repository import TaskRepository


class CreateTaskUseCase:
    def __init__(self, task_repository: TaskRepository) -> None:
        self._tasks = task_repository

    def execute(self, dto: TaskCreate, user_id: UUID) -> TaskResponse:
        task = Task(title=dto.title, description=dto.description, user_id=user_id)
        self._tasks.create(task)
        return TaskResponse.from_entity(task)
```

> El `user_id` llega como argumento escalar (no dentro del DTO): lo aporta el
> router desde el JWT, nunca desde el body de la request.

#### `src/application/use_cases/list_my_tasks.py`

```python
from uuid import UUID

from src.application.dto.task_dto import TaskListResponse, TaskResponse
from src.domain.ports.task_repository import TaskRepository


class ListMyTasksUseCase:
    def __init__(self, task_repository: TaskRepository) -> None:
        self._tasks = task_repository

    def execute(self, user_id: UUID, offset: int = 0, limit: int = 100) -> TaskListResponse:
        limit = max(1, min(limit, 200))
        offset = max(0, offset)
        tasks = self._tasks.list_for_user(user_id, offset=offset, limit=limit)
        return TaskListResponse(
            items=[TaskResponse.from_entity(task) for task in tasks],
            total=self._tasks.count_for_user(user_id),
            offset=offset,
            limit=limit,
        )
```

#### `src/application/use_cases/delete_task.py` (con regla de propiedad)

```python
from uuid import UUID

from src.domain.entities.task import Task
from src.domain.errors import TaskNotFoundError
from src.domain.ports.task_repository import TaskRepository


class DeleteTaskUseCase:
    def __init__(self, task_repository: TaskRepository) -> None:
        self._tasks = task_repository

    def execute(self, task_id: UUID, owner_id: UUID) -> None:
        task = self._get_owned(task_id, owner_id)
        self._tasks.delete(task.id)

    def _get_owned(self, task_id: UUID, owner_id: UUID) -> Task:
        task = self._tasks.get_by_id(task_id)
        if task is None or task.user_id != owner_id:
            raise TaskNotFoundError(str(task_id))
        return task
```

> La regla de negocio vive aquí: una tarea ajena responde **404** (misma
> respuesta que una inexistente), evitando filtrar la existencia de recursos
> de otros usuarios.

#### `src/application/use_cases/mark_task_done.py`

```python
from uuid import UUID

from src.application.dto.task_dto import TaskResponse
from src.domain.errors import TaskNotFoundError
from src.domain.ports.task_repository import TaskRepository


class MarkTaskDoneUseCase:
    def __init__(self, task_repository: TaskRepository) -> None:
        self._tasks = task_repository

    def execute(self, task_id: UUID, owner_id: UUID) -> TaskResponse:
        task = self._tasks.get_by_id(task_id)
        if task is None or task.user_id != owner_id:
            raise TaskNotFoundError(str(task_id))
        updated = self._tasks.set_done(task_id, done=True)
        return TaskResponse.from_entity(updated)
```

---

## Paso 3 · Infraestructura

### 3.1 Modelo — `src/infrastructure/database/models.py` (añade)

```python
class TaskModel(Base):
    __tablename__ = "tasks"

    id: Mapped[str] = mapped_column(String(36), primary_key=True)
    title: Mapped[str] = mapped_column(String(100))
    description: Mapped[str] = mapped_column(String(500), default="")
    done: Mapped[bool] = mapped_column(Boolean, default=False)
    user_id: Mapped[str] = mapped_column(
        String(36), ForeignKey("users.id", ondelete="CASCADE"), index=True
    )
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), default=utcnow, onupdate=utcnow
    )
```

Importa arriba del archivo (ya están la mayoría): `Boolean, DateTime, ForeignKey,
String` de `sqlalchemy` y `Mapped, mapped_column` de `sqlalchemy.orm`.

### 3.2 Mapper — `src/infrastructure/database/mappers.py` (añade)

```python
def task_to_entity(model: TaskModel) -> Task:
    return Task(
        id=UUID(model.id),
        title=model.title,
        description=model.description,
        done=model.done,
        user_id=UUID(model.user_id),
        created_at=as_utc(model.created_at),
        updated_at=as_utc(model.updated_at),
    )


def task_to_model(entity: Task) -> TaskModel:
    return TaskModel(
        id=str(entity.id),
        title=entity.title,
        description=entity.description,
        done=entity.done,
        user_id=str(entity.user_id),
        created_at=entity.created_at,
        updated_at=entity.updated_at,
    )
```

### 3.3 Repositorio — `src/infrastructure/repositories/sqlalchemy_task_repository.py`

```python
from uuid import UUID

from sqlalchemy import func, select

from src.domain.entities.task import Task
from src.domain.errors import TaskNotFoundError
from src.domain.ports.task_repository import TaskRepository
from src.infrastructure.database.mappers import task_to_entity, task_to_model
from src.infrastructure.database.models import TaskModel
from src.infrastructure.repositories.base import SqlAlchemyRepository


class SqlAlchemyTaskRepository(SqlAlchemyRepository, TaskRepository):
    def create(self, task: Task) -> None:
        with self._session(commit=True) as session:
            session.add(task_to_model(task))

    def get_by_id(self, task_id: UUID) -> Task | None:
        with self._session() as session:
            model = session.get(TaskModel, str(task_id))
            return task_to_entity(model) if model else None

    def list_for_user(
        self, user_id: UUID, offset: int = 0, limit: int = 100
    ) -> list[Task]:
        with self._session() as session:
            models = session.scalars(
                select(TaskModel)
                .where(TaskModel.user_id == str(user_id))
                .order_by(TaskModel.created_at.desc())
                .offset(offset)
                .limit(limit)
            ).all()
            return [task_to_entity(model) for model in models]

    def count_for_user(self, user_id: UUID) -> int:
        with self._session() as session:
            return int(
                session.scalar(
                    select(func.count())
                    .select_from(TaskModel)
                    .where(TaskModel.user_id == str(user_id))
                )
                or 0
            )

    def set_done(self, task_id: UUID, done: bool) -> Task:
        with self._session(commit=True) as session:
            model = session.get(TaskModel, str(task_id))
            if not model:
                raise TaskNotFoundError(str(task_id))
            model.done = done
            session.flush()
            return task_to_entity(model)

    def delete(self, task_id: UUID) -> None:
        with self._session(commit=True) as session:
            model = session.get(TaskModel, str(task_id))
            if not model:
                raise TaskNotFoundError(str(task_id))
            session.delete(model)
```

### 3.4 Migración

```bash
.venv/bin/alembic revision --autogenerate -m "add tasks table"
```

Revisa el archivo generado en `alembic/versions/`; debe contener
aproximadamente:

```python
def upgrade() -> None:
    op.create_table(
        "tasks",
        sa.Column("id", sa.String(length=36), nullable=False),
        sa.Column("title", sa.String(length=100), nullable=False),
        sa.Column("description", sa.String(length=500), nullable=False),
        sa.Column("done", sa.Boolean(), nullable=False),
        sa.Column("user_id", sa.String(length=36), nullable=False),
        sa.Column("created_at", sa.DateTime(timezone=True), nullable=False),
        sa.Column("updated_at", sa.DateTime(timezone=True), nullable=False),
        sa.ForeignKeyConstraint(["user_id"], ["users.id"], ondelete="CASCADE"),
        sa.PrimaryKeyConstraint("id"),
    )
    op.create_index(op.f("ix_tasks_user_id"), "tasks", ["user_id"], unique=False)


def downgrade() -> None:
    op.drop_index(op.f("ix_tasks_user_id"), table_name="tasks")
    op.drop_table("tasks")
```

> Ajusta `nullable`/defaults según lo que alembic infiera de tus `default=` de
> Python (SQLAlchemy los aplica en la app, no en la BD: si la columna debe ser
> `NOT NULL` con default real, márcalo en el modelo). Aplica con:

```bash
.venv/bin/alembic upgrade head
```

---

## Paso 4 · Interfaces

### 4.1 Providers — `src/interfaces/api/dependencies/providers.py` (añade)

```python
from src.application.use_cases.create_task import CreateTaskUseCase
from src.application.use_cases.delete_task import DeleteTaskUseCase
from src.application.use_cases.list_my_tasks import ListMyTasksUseCase
from src.application.use_cases.mark_task_done import MarkTaskDoneUseCase
from src.domain.ports.task_repository import TaskRepository
from src.infrastructure.repositories.sqlalchemy_task_repository import (
    SqlAlchemyTaskRepository,
)


def get_task_repository() -> TaskRepository:
    return SqlAlchemyTaskRepository(get_database())


def get_create_task_use_case() -> CreateTaskUseCase:
    return CreateTaskUseCase(task_repository=get_task_repository())


def get_list_my_tasks_use_case() -> ListMyTasksUseCase:
    return ListMyTasksUseCase(task_repository=get_task_repository())


def get_mark_task_done_use_case() -> MarkTaskDoneUseCase:
    return MarkTaskDoneUseCase(task_repository=get_task_repository())


def get_delete_task_use_case() -> DeleteTaskUseCase:
    return DeleteTaskUseCase(task_repository=get_task_repository())
```

### 4.2 Router — `src/interfaces/api/routers/tasks_router.py`

```python
from uuid import UUID

from fastapi import APIRouter, Depends

from src.application.dto.task_dto import (
    TaskCreate,
    TaskListResponse,
    TaskResponse,
)
from src.application.use_cases.create_task import CreateTaskUseCase
from src.application.use_cases.delete_task import DeleteTaskUseCase
from src.application.use_cases.list_my_tasks import ListMyTasksUseCase
from src.application.use_cases.mark_task_done import MarkTaskDoneUseCase
from src.domain.entities.user import User
from src.interfaces.api.dependencies.auth import get_current_user
from src.interfaces.api.dependencies.providers import (
    get_create_task_use_case,
    get_delete_task_use_case,
    get_list_my_tasks_use_case,
    get_mark_task_done_use_case,
)

router = APIRouter(prefix="/tasks", tags=["tasks"])


@router.get("", response_model=TaskListResponse)
def list_my_tasks(
    offset: int = 0,
    limit: int = 100,
    user: User = Depends(get_current_user),
    use_case: ListMyTasksUseCase = Depends(get_list_my_tasks_use_case),
) -> TaskListResponse:
    return use_case.execute(user_id=user.id, offset=offset, limit=limit)


@router.post("", response_model=TaskResponse, status_code=201)
def create_task(
    dto: TaskCreate,
    user: User = Depends(get_current_user),
    use_case: CreateTaskUseCase = Depends(get_create_task_use_case),
) -> TaskResponse:
    return use_case.execute(dto, user_id=user.id)


@router.patch("/{task_id}/done", response_model=TaskResponse)
def mark_task_done(
    task_id: UUID,
    user: User = Depends(get_current_user),
    use_case: MarkTaskDoneUseCase = Depends(get_mark_task_done_use_case),
) -> TaskResponse:
    return use_case.execute(task_id, owner_id=user.id)


@router.delete("/{task_id}", status_code=204)
def delete_task(
    task_id: UUID,
    user: User = Depends(get_current_user),
    use_case: DeleteTaskUseCase = Depends(get_delete_task_use_case),
) -> None:
    use_case.execute(task_id, owner_id=user.id)
```

> El `user_id`/`owner_id` **nunca viene del cliente**: se toma del JWT vía
> `get_current_user`. Así no se puede manipular por JSON.

### 4.3 Errores — `src/interfaces/api/exception_handlers.py` (añade)

Importa `TaskNotFoundError` arriba, junto a los demás, y registra el handler
**dentro de** `register_exception_handlers(app)`:

```python
@app.exception_handler(TaskNotFoundError)
def _task_not_found(request: Request, exc: TaskNotFoundError) -> JSONResponse:
    return _error_response(404, str(exc))
```

### 4.4 Registro — `src/interfaces/api/app.py`

```python
from src.interfaces.api.routers import auth_router, tasks_router, users_router
# ...
app.include_router(auth_router.router)
app.include_router(users_router.router)
app.include_router(tasks_router.router)
```

---

## Paso 5 · Tests

### 5.1 Unit — `tests/unit/test_task_use_cases.py`

```python
from types import SimpleNamespace
from uuid import uuid4

import pytest

from src.application.dto.task_dto import TaskCreate
from src.application.use_cases.create_task import CreateTaskUseCase
from src.application.use_cases.delete_task import DeleteTaskUseCase
from src.application.use_cases.list_my_tasks import ListMyTasksUseCase
from src.domain.errors import TaskNotFoundError
from src.infrastructure.config import Settings
from src.infrastructure.database.session import Database
from src.infrastructure.repositories.sqlalchemy_task_repository import (
    SqlAlchemyTaskRepository,
)


@pytest.fixture
def env():
    settings = Settings(_env_file=None, secret_key="unit-secret")
    database = Database(settings)
    database.create_all()
    tasks = SqlAlchemyTaskRepository(database)
    owner = uuid4()
    return SimpleNamespace(
        database=database,
        tasks=tasks,
        owner=owner,
        create=CreateTaskUseCase(tasks),
        list_my=ListMyTasksUseCase(tasks),
        delete=DeleteTaskUseCase(tasks),
    )


def test_create_task_persists_for_owner(env):
    response = env.create.execute(TaskCreate(title="Comprar leche"), env.owner)
    stored = env.tasks.get_by_id(response.id)
    assert stored.title == "Comprar leche"
    assert stored.user_id == env.owner
    assert stored.done is False


def test_list_only_returns_own_tasks(env):
    other = uuid4()
    env.create.execute(TaskCreate(title="mía"), env.owner)
    env.create.execute(TaskCreate(title="ajena"), other)

    result = env.list_my.execute(env.owner)
    assert result.total == 1
    assert result.items[0].title == "mía"


def test_delete_foreign_task_raises_not_found(env):
    foreign = env.create.execute(TaskCreate(title="ajena"), uuid4())
    with pytest.raises(TaskNotFoundError):
        env.delete.execute(foreign.id, env.owner)


def test_delete_unknown_task_raises(env):
    with pytest.raises(TaskNotFoundError):
        env.delete.execute(uuid4(), env.owner)
```

### 5.2 Integración — `tests/integration/test_tasks_api.py`

```python
def test_requires_auth(client):
    assert client.get("/tasks").status_code == 401


def test_create_and_list_task(client, create_user, login):
    create_user("alice")
    headers, _ = login("alice", "password1")

    created = client.post(
        "/tasks", headers=headers, json={"title": "Escribir docs"}
    )
    assert created.status_code == 201
    assert created.json()["title"] == "Escribir docs"

    listing = client.get("/tasks", headers=headers)
    assert listing.status_code == 200
    body = listing.json()
    assert body["total"] == 1
    assert body["items"][0]["id"] == created.json()["id"]


def test_empty_title_returns_422(client, create_user, login):
    create_user("alice")
    headers, _ = login("alice", "password1")
    response = client.post("/tasks", headers=headers, json={"title": ""})
    assert response.status_code == 422


def test_delete_foreign_task_returns_404(client, create_user, login):
    create_user("alice")
    create_user("bob")
    alice_headers, _ = login("alice", "password1")
    bob_headers, _ = login("bob", "password1")

    created = client.post(
        "/tasks", headers=bob_headers, json={"title": "de bob"}
    ).json()

    response = client.delete(f"/tasks/{created['id']}", headers=alice_headers)
    assert response.status_code == 404


def test_delete_own_task_returns_204(client, create_user, login):
    create_user("alice")
    headers, _ = login("alice", "password1")
    created = client.post("/tasks", headers=headers, json={"title": "tarea"})

    response = client.delete(f"/tasks/{created.json()['id']}", headers=headers)
    assert response.status_code == 204
    assert client.get("/tasks", headers=headers).json()["total"] == 0


def test_mark_done_flow(client, create_user, login):
    create_user("alice")
    headers, _ = login("alice", "password1")
    created = client.post("/tasks", headers=headers, json={"title": "tarea"})

    response = client.patch(
        f"/tasks/{created.json()['id']}/done", headers=headers
    )
    assert response.status_code == 200
    assert response.json()["done"] is True
```

---

## Paso 6 · Verificación

```bash
.venv/bin/pytest
.venv/bin/ruff check src tests main.py
.venv/bin/mypy src
```

## Paso 7 · Documentación

Añadir al `README.md`:

```markdown
| GET | `/tasks` | Bearer |
| POST | `/tasks` | Bearer |
| PATCH | `/tasks/{id}/done` | Bearer |
| DELETE | `/tasks/{id}` | Bearer |
```

---

## Checklist de este módulo

- [x] Entidad + puerto en `domain/`
- [x] `TaskNotFoundError` en `domain/errors.py`
- [x] DTOs con `from_entity` y `TaskListResponse`
- [x] 4 casos de uso con inyección por constructor
- [x] `TaskModel` en `models.py` con FK + index
- [x] Mappers con `as_utc`
- [x] `SqlAlchemyTaskRepository` con `_session(commit=True)`
- [x] Migración generada, revisada y aplicada
- [x] Providers `get_*_use_case`
- [x] Router con `get_current_user` (sin confiar en `user_id` del body)
- [x] Handler `TaskNotFoundError → 404`
- [x] `app.include_router(tasks_router.router)`
- [x] Tests unit (reglas de propiedad) + integración (401/422/404/204)
- [x] `pytest` + `ruff` + `mypy` en verde
- [x] Fila(s) añadidas a la tabla del `README.md`

Volver: [04-checklist.md](04-checklist.md) para la revisión general.
