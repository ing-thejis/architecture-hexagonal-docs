# 04 · Checklist de implementación

Copia esta lista en la descripción de tu PR y marca lo que aplique.

## Dominio (`src/domain/`)

- [ ] Entidad como `@dataclass(slots=True)` con `id: UUID` autogenerado y timestamps UTC (`field(default_factory=...)`).
- [ ] Entidad sin imports de SQLAlchemy/FastAPI/Pydantic (Python puro).
- [ ] Enums como `StrEnum` con `.parse()` si la BD guarda strings.
- [ ] Errores nuevos en `errors.py`, heredando de `DomainError` (o subtipo como `TokenError`), con mensaje construido en el `__init__`.
- [ ] Puerto `ABC` en `ports/<entidad>_repository.py` con `@abstractmethod` por operación, devolviendo entidades (nunca modelos ORM).
- [ ] Sin `password_hash` u otros secretos pensados para respuestas (se filtran en los DTOs).

## Aplicación (`src/application/`)

- [ ] DTOs request (`XCreate`/`XRequest`) con `Field(min_length/max_length/pattern)`.
- [ ] DTO response (`XResponse`) con `from_entity()` classmethod.
- [ ] `XListResponse` con `items`, `total`, `offset`, `limit` si hay listado.
- [ ] Un archivo por caso de uso (`<verbo>_<sustantivo>.py`), clase `XUseCase` con `execute()`.
- [ ] Puertos inyectados por `__init__`; sin imports de `infrastructure`/`interfaces`.
- [ ] Paginación acotada en el caso de uso (`limit = max(1, min(limit, 200))`).
- [ ] Reglas de negocio (propiedad, unicidad, transiciones) validadas aquí, lanzando errores de dominio.

## Infraestructura (`src/infrastructure/`)

- [ ] Modelo añadido **en `database/models.py`** (no en archivo nuevo: `alembic/env.py` solo importa ese módulo).
- [ ] PK `String(36)`; FK `String(36)` + `ForeignKey(..., ondelete=...)` + `index=True`.
- [ ] `CheckConstraint` con nombre `ck_<tabla>_<col>` si hay enum guardado como string.
- [ ] Fechas: `DateTime(timezone=True)`, `default=utcnow`, `onupdate=utcnow` en `updated_at`.
- [ ] Mappers `x_to_entity`/`x_to_model` en `mappers.py`, con `as_utc()` en toda fecha leída y `UUID(...)`/`str(...)` en IDs.
- [ ] Repositorio hereda `SqlAlchemyRepository` + el puerto; lecturas con `self._session()`, escrituras con `self._session(commit=True)`.
- [ ] Unicidad validada antes de `add` → `XAlreadyExistsError`; ID inexistente en escritura → `XNotFoundError`.
- [ ] Migración: `alembic revision --autogenerate` → diff revisado a mano → `alembic upgrade head`.

## Interfaces (`src/interfaces/api/`)

- [ ] Provider `get_<x>_use_case()` (+ `get_<x>_repository()` si hace falta) en `dependencies/providers.py`.
- [ ] `@lru_cache(maxsize=1)` solo en objetos de ciclo de vida largo (database, hasher, token provider).
- [ ] Router nuevo en `routers/<x>_router.py` con `APIRouter(prefix=..., tags=[...])`.
- [ ] `response_model` explícito en todos los endpoints; `status_code=201` en POST, `204` en DELETE.
- [ ] Auth por `Depends(get_current_user)` o `Depends(require_roles(...))` donde aplique; sin reimplementar JWT.
- [ ] IDs/owners identitarios tomados del JWT, nunca del body.
- [ ] `HTTPException` solo para asuntos de transporte (400/401/403), no para errores de negocio.
- [ ] Errores de dominio nuevos registrados en `exception_handlers.py` con su status (404/409/…).
- [ ] `app.include_router(...)` añadido en `app.py`.

## Tests

- [ ] Unit: fixture con `Database(settings)` + `create_all()` + repos reales (patrón `env`), sin mocks.
- [ ] Unit: happy path + camino de error (not-found/duplicado) + regla de negocio por operación.
- [ ] Integración: fixtures `client`, `admin_headers`/`login`/`create_user` de `tests/conftest.py`.
- [ ] Integración por endpoint: 401 sin token, 403 rol incorrecto (si aplica), 422 validación, 404 inexistente, 409 duplicado (si aplica), happy path con shape del body.
- [ ] Ningún test modifica fixtures globales (`db_reset` autouse se mantiene).

## Verificación

```bash
.venv/bin/pytest
.venv/bin/ruff check src tests main.py
.venv/bin/mypy src
```

- [ ] Los tres comandos en verde.

## Documentación

- [ ] Endpoints añadidos a la tabla del [`README.md`](../README.md).
- [ ] Variables de entorno nuevas en `.env.example` y en *Environment Variables* de `AGENTS.md`.
- [ ] Cambios de flujo/setup reflejados en `AGENTS.md` (*Commands* / *Gotchas*).

---

## Comandos de referencia

| Tarea | Comando |
|---|---|
| Tests (todos) | `.venv/bin/pytest` |
| Tests unit | `.venv/bin/pytest tests/unit` |
| Tests integración | `.venv/bin/pytest tests/integration` |
| Lint | `.venv/bin/ruff check src tests main.py` |
| Typecheck | `.venv/bin/mypy src` |
| Nueva migración | `.venv/bin/alembic revision --autogenerate -m "descripción"` |
| Aplicar migraciones | `.venv/bin/alembic upgrade head` |
| Servidor dev | `.venv/bin/uvicorn main:app --reload` |
| Seed admin | `.venv/bin/python scripts/seed_admin.py` |

---

## Gotchas del proyecto

1. **Python 3.14+** requerido (`pyproject.toml`); usa siempre el `.venv` del repo.
2. **UUIDs en SQLite**: la PK es `String(36)`; todo `str(uuid)` al persistir y `UUID(...)` al leer.
3. **SQLite pierde `tzinfo`**: toda fecha leída por un mapper pasa por `as_utc()`.
4. **`models.py` es único punto de registro de tablas**: modelos fuera de ese archivo son invisibles para `alembic/env.py` y para `create_all()` de los tests.
5. **Migraciones al arrancar**: la app las ejecuta en el lifespan (`run_migrations()`); no hace falta correrlas a mano en dev, pero sí revisar el diff autogenerado.
6. **Foreign keys en SQLite**: activadas por `PRAGMA foreign_keys=ON` en `session.py`; respeta los `ondelete`.
7. **Límites de paginación**: clamp en el caso de uso (1–200), no en el router.
8. **Errores de dominio → HTTP**: si no registras el handler, todo cae a 400 (fallback `DomainError`).
9. **`HTTPException` solo en `interfaces/`**: dominio y aplicación jamás la importan.
10. **Endpoints síncronos**: usa `def`, no `async def` (SQLAlchemy sync).
11. **Ruff**: `E501` y `B008` ignorados, line-length 100, imports con `known-first-party = ["src"]`; `alembic/versions` está excluido del lint.
12. **Tests**: `BCRYPT_ROUNDS=4` y `SQLITE_URL` temporal los fija `tests/conftest.py` antes de importar la app; no los cambies en tests.
