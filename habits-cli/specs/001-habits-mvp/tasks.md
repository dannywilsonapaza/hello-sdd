# Tareas de Implementación — Spec 001 (MVP de habits-cli)

**Estado:** Aprobado para ejecución  
**Versión:** 1.1  
**Referencias:** [Constitution](file:///c:/cursos-dev/hello-sdd/habits-cli/docs/constitution.md), [Spec 001 v2.1](file:///c:/cursos-dev/hello-sdd/habits-cli/specs/001-habits-mvp/spec.md) y [Plan Técnico v1.0](file:///c:/cursos-dev/hello-sdd/habits-cli/specs/001-habits-mvp/plan.md)

---

## Fase 1: Estructura Base del Proyecto y Entorno de Pruebas

- [ ] **Tarea 1.1: Inicialización de paquetes `habits` y `tests`**
  - **RF / Casos cubiertos:** `RNF-2`
  - **Descripción:** Crear los paquetes `habits/` y `tests/` con sus respectivos `__init__.py`, `habits/__main__.py` y `habits/cli.py` (esqueleto inicial).
  - **Hecho cuando:** `python -m habits` es invocable sin error de módulo y `pytest -q` detecta el directorio `tests/` sin errores de importación.

---

## Fase 2: Núcleo Puro (`habits/core.py`) y Pruebas Unitarias

- [ ] **Tarea 2.1: Normalización, clave de identidad y validación de nombres**
  - **RF / Casos cubiertos:** `RF-1`, `RF-3`, `RF-4`, `CL-7`, `CL-8`, `CL-9`, `CL-10`, `CL-22`, `CL-23`
  - **Descripción:** Implementar en `habits/core.py` las funciones puras `normalize_name(raw_name: str) -> str`, `to_identity_key(normalized_name: str) -> str` (`casefold`) y `validate_name(normalized_name: str) -> None`, junto con la excepción `ValidationError`.
  - **Hecho cuando:** `pytest -q tests/test_core.py -k "test_normalize or test_identity or test_validate"` pasa al 100%, validando:
    * Colapso de espacios múltiples y tabuladores `\t` en un solo espacio (`CL-7`, `CL-8`).
    * Clave insensible a mayúsculas y sensible a acentos (`CL-9`, `CL-10`).
    * Nombre de exactamente 100 caracteres aceptado (`CL-22`).
    * Rechazo con `ValidationError` si está vacío, supera 100 caracteres (`CL-23`) o contiene caracteres de control (`\n`, `\r`, `\v`, `\f`, `\0`).

- [ ] **Tarea 2.2: Algoritmo de cálculo de racha y formato de texto**
  - **RF / Casos cubiertos:** `RF-9`, `RF-15`, `CL-2`, `CL-3`, `CL-4`, `CL-5`, `CL-6`, `CL-11`, `CL-12`
  - **Descripción:** Implementar en `habits/core.py` las funciones puras `calculate_streak(completed_dates: set[datetime.date], today: datetime.date) -> int` y `format_streak_text(count: int) -> str`.
  - **Hecho cuando:** `pytest -q tests/test_core.py -k "test_streak or test_format_streak"` pasa al 100%, validando:
    * Sin marcas -> racha 0 (`CL-5`).
    * Marcado hoy -> racha 1 (`CL-2`).
    * Marcado ayer pero no hoy -> racha activa hasta ayer (`CL-3`).
    * Marcado anteayer, ni ayer ni hoy -> racha 0 (`CL-4`).
    * Racha antigua rota y marcada hoy -> racha 1 sin revivir la cadena anterior (`CL-6`).
    * Continuidad en transiciones de fin de mes y fin de año (`CL-11`).
    * Idempotencia ante fechas duplicadas o desordenadas (`CL-12`).
    * Formato de concordancia: `0 días`, `1 día`, `2 días` (`RF-15`).

- [ ] **Tarea 2.3: Operaciones de dominio sobre hábitos y ordenación de listado**
  - **RF / Casos cubiertos:** `RF-2`, `RF-3`, `RF-5`, `RF-6`, `RF-7`, `RF-8`
  - **Descripción:** Implementar en `habits/core.py` las funciones puras `add_habit(habits_data: dict, name: str) -> tuple[dict, str]`, `mark_habit_done(habits_data: dict, name: str, today: datetime.date) -> tuple[dict, bool, str]` y `list_habits(habits_data: dict, today: datetime.date) -> list[tuple[str, int]]`, con excepciones `DuplicateHabitError` y `HabitNotFoundError`.
  - **Hecho cuando:** `pytest -q tests/test_core.py -k "test_add_habit or test_mark_habit or test_list_habits"` pasa al 100%, validando creación de hábitos, rechazo de duplicados por identidad, marca de hoy (con bandera indicando si ya estaba marcado) y listado ordenado por punto de código Unicode.

---

## Fase 3: Persistencia JSON (`habits/storage.py`) y Pruebas de Almacenamiento

- [ ] **Tarea 3.1: Resolución de ruta y soporte de variable `HABITS_FILE`**
  - **RF / Casos cubiertos:** `RF-11`
  - **Descripción:** Implementar en `habits/storage.py` la función `get_storage_path() -> Path`, priorizando `HABITS_FILE` y con fallback a `~/.habits.json`.
  - **Hecho cuando:** `pytest -q tests/test_storage.py -k "test_get_storage_path"` pasa al 100%, verificando la lectura de la variable de entorno y la ruta por defecto en home.

- [ ] **Tarea 3.2: Carga, validación de esquema e integridad de datos**
  - **RF / Casos cubiertos:** `RF-11`, `RF-14`, `CL-13`, `CL-14`, `CL-15`, `CL-17`, `CL-24`
  - **Descripción:** Implementar en `habits/storage.py` la función `load_habits(path: Path | None, today: datetime.date | None) -> dict`, con excepciones `CorruptDataError`, `InconsistentDateError` y `StoragePermissionError`.
  - **Hecho cuando:** `pytest -q tests/test_storage.py -k "test_load_habits"` pasa al 100%, validando:
    * Archivo no existente -> retorna estructura vacía sin error (`CL-13`).
    * Archivo de 0 bytes -> retorna estructura vacía sin error (`CL-14`).
    * JSON corrupto o esquema inválido -> eleva `CorruptDataError` (`CL-15`).
    * Fechas no ISO o días de calendario imposibles (`2026-02-30`) -> eleva `CorruptDataError` (`CL-24`).
    * Fechas futuras respecto a `today` -> eleva `InconsistentDateError` (`CL-17`).
    * Discrepancia entre clave de diccionario y `casefold(name)` -> eleva `CorruptDataError`.

- [ ] **Tarea 3.3: Guardado atómico y autocreación de directorios**
  - **RF / Casos cubiertos:** `RF-11`, `RF-12`, `CL-19`, `CL-25`
  - **Descripción:** Implementar en `habits/storage.py` la función `save_habits(habits_data: dict, path: Path | None)` con creación de directorios padres faltantes (`os.makedirs`) y volcado atómico con archivo temporal + `os.replace`.
  - **Hecho cuando:** `pytest -q tests/test_storage.py -k "test_save_habits"` pasa al 100%, validando creación de carpetas padres inexistentes (`CL-25`), escritura del archivo JSON y atomicidad del reemplazo (`CL-19`).

---

## Fase 4: Capa de Interfaz de Línea de Comandos (`habits/cli.py`) y Tests de Integración

- [ ] **Tarea 4.1: Comando `add` en CLI con argumentos flexibles**
  - **RF / Casos cubiertos:** `RF-1`, `RF-2`, `RF-3`, `RF-4`, `RF-13`, `RF-15`
  - **Descripción:** Implementar en `habits/cli.py` el handler para `add` soportando argumentos con o sin comillas (`nargs="+"`), conectando con `storage` y `core`.
  - **Hecho cuando:** `pytest -q tests/test_cli.py -k "test_cli_add"` pasa al 100%, validando:
    * Creación exitosa: salida por `stdout` con nombre mostrado y exit code `0`.
    * Duplicados o nombres inválidos: mensaje claro en español por `stderr` con exit code `1`.
    * Argumentos faltantes: mensaje por `stderr` con exit code `2`.

- [ ] **Tarea 4.2: Comando `done` en CLI con idempotencia**
  - **RF / Casos cubiertos:** `RF-5`, `RF-6`, `RF-7`, `RF-13`, `RF-15`, `CL-1`
  - **Descripción:** Implementar en `habits/cli.py` el handler para `done` soportando `nargs="+"`.
  - **Hecho cuando:** `pytest -q tests/test_cli.py -k "test_cli_done"` pasa al 100%, validando:
    * Primera marca de hoy: confirmación por `stdout` y exit code `0`.
    * Segunda marca de hoy: mensaje informativo por `stdout` y exit code `0` (`CL-1`).
    * Hábito inexistente: mensaje de error por `stderr` y exit code `1`.

- [ ] **Tarea 4.3: Comando `list` en CLI con formato y lista vacía**
  - **RF / Casos cubiertos:** `RF-8`, `RF-10`, `RF-13`, `RF-15`, `CL-21`
  - **Descripción:** Implementar en `habits/cli.py` el handler para `list`.
  - **Hecho cuando:** `pytest -q tests/test_cli.py -k "test_cli_list"` pasa al 100%, validando:
    * Con hábitos: salida `<nombre>: <N> días` en `stdout` ordenada por Unicode y exit code `0`.
    * Sin hábitos: mensaje informativo en español por `stdout` y exit code `0`.
    * Argumentos adicionales sobrantes: error por `stderr` y exit code `2` (`CL-21`).

- [ ] **Tarea 4.4: Invocación global, ayuda, captura de fecha y captura de excepciones**
  - **RF / Casos cubiertos:** `RF-13`, `RF-14`, `CL-16`, `CL-18`, `CL-20`
  - **Descripción:** Implementar en `habits/cli.py` la captura única de `today` al inicio (`D-4`, `CL-18`), ayuda en español por `stderr` al invocar sin argumentos (código `2`, `CL-20`), manejo de comandos no reconocidos (código `2`), y captura global de excepciones de persistencia/permisos/corrupción por `stderr` (código `1`, `CL-16`) sin imprimir trazas de excepción (`Traceback`).
  - **Hecho cuando:** `pytest -q tests/test_cli.py -k "test_cli_global or test_cli_errors"` pasa al 100%, confirmando ausencia total de trazas de excepción.

---

## Fase 5: Validación Integral y Criterios de Finalización

- [ ] **Tarea 5.1: Ejecución completa de la suite de pruebas y verificación de casos límite**
  - **RF / Casos cubiertos:** `RF-1` al `RF-15`, `RNF-1` al `RNF-6`, `CL-1` al `CL-25`, Principio 4 de la Constitución
  - **Descripción:** Ejecutar la suite completa de pruebas unitarias, de almacenamiento y de integración CLI.
  - **Hecho cuando:** `pytest -q` pasa al 100% (verde) con todos los tests pasando y 0 fallos, cubriendo íntegramente los 25 casos límite y los criterios de aceptación del MVP.
