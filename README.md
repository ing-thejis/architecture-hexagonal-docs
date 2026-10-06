# Documentación

Guías del proyecto para mantener la arquitectura hexagonal consistente.

## Índice

| #   | Documento                                                    | Contenido                                                                                                                                                                                             |
| --- | ------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | [01-arquitectura.md](01-arquitectura.md)                     | Visión general de la arquitectura: capas, regla de dependencias, flujo de una request, manejo de errores y convenciones globales de naming y tipos.                                                   |
| 2   | [02-guia-paso-a-paso.md](02-guia-paso-a-paso.md)             | **Guía principal**: los pasos 0–7 para implementar cualquier módulo nuevo, con explicación y snippet de código por capa (dominio → aplicación → infraestructura → interfaces → tests → verificación). |
| 3   | [03-ejemplo-completo-tasks.md](03-ejemplo-completo-tasks.md) | Módulo ficticio `tasks` desarrollado end-to-end: todos los archivos en orden de creación, desde la entidad hasta los tests, listo para copiar y adaptar.                                              |
| 4   | [04-checklist.md](04-checklist.md)                           | Checklist por capa para revisar antes de hacer merge, tabla de comandos de verificación y lista de gotchas del proyecto.                                                                              |

## Cómo usar esta documentación

- **Primera vez implementando un módulo**: lee el orden `01 → 02 → 03`, y usa `04` como repaso final.
- **Ya conoces la arquitectura**: ve directo a `02-guia-paso-a-paso.md` y usa `03` como plantilla de copiar/pegar.
- **Revisión de código**: usa `04-checklist.md`.

La guía toma como ejemplo canónico el módulo existente de `users`/`auth`
(`src/domain/entities/user.py` → `src/interfaces/api/routers/users_router.py`),
y desarrolla un módulo ficticio completo llamado `tasks` en `03` sin crear
archivos reales en `src/`.

## Requisitos previos

- Python 3.14+ y el entorno configurado (ver [README](../README.md)).
- Familiaridad con FastAPI, Pydantic v2 y SQLAlchemy 2.0.
