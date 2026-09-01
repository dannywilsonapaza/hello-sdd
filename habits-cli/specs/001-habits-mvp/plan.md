# Plan Técnico de Implementación — Spec 001 (MVP de habits-cli)

**Estado:** Aprobado para implementación  
**Versión del Plan:** 1.0  
**Referencia:** [Constitution](file:///c:/cursos-dev/hello-sdd/habits-cli/docs/constitution.md) y [Spec 001 v2.1](file:///c:/cursos-dev/hello-sdd/habits-cli/specs/001-habits-mvp/spec.md)

---

## 1. Estructura de Módulos y Arquitectura

En estricto cumplimiento con los principios 1, 3, 5 y 6 de la Constitución, el código se estructurará dividiendo la lógica pura del dominio, la persistencia en disco y la capa de interfaz de línea de comandos.

```text
habits-cli/
├── habits/
│   ├── __init__.py         # Metadatos del paquete (__version__)
│   ├── __main__.py         # Punto de entrada para `python -m habits`
│   ├── core.py             # Dominio puro (cálculos, validaciones, transformaciones sin I/O)
│   ├── storage.py          # Persistencia JSON local, schema validation y escritura atómica
│   └── cli.py              # Parsing con argparse, orquestación, formato de texto, stdout/stderr
├── tests/
│   ├── __init__.py
│   ├── test_core.py        # Tests unitarios del núcleo puro
│   ├── test_storage.py     # Tests de persistencia, concurrencia/atomicidad y corrupción
│   └── test_cli.py         # Tests de integración CLI end-to-end (argparse, exit codes, streams)
├── docs/
│   └── constitution.md
└── specs/
    └── 001-habits-mvp/
        ├── spec.md
        └── plan.md
```

### Responsabilidades por Módulo

#### `habits/core.py` (Núcleo Puro) `[Cubre RF-1, RF-2, RF-3, RF-4, RF-5, RF-6, RF-7, RF-8, RF-9, RF-15]`
* **Regla constitucional (P3):** No importa `argparse`, `sys`, `pathlib`, no hace E/S ni ejecuta `print()`. Recibe estructuras de datos y retorna estructuras de datos o eleva excepciones de dominio.
* **Funciones públicas principales:**
  * `normalize_name(raw_name: str) -> str`: Limpia espacios iniciales/finales y colapsa espacios internos y `\t`. `[RF-1, D-1]`
  * `validate_name(normalized_name: str) -> None`: Valida que no esté vacío, longitud $\le 100$ y ausencia de caracteres de control (`\n`, `\r`, `\v`, `\f`, `\0`). `[RF-4, CL-22, CL-23]`
  * `to_identity_key(normalized_name: str) -> str`: Retorna el nombre procesado con `str.casefold()`. `[RF-3, D-3]`
  * `add_habit(habits_data: dict, name: str) -> tuple[dict, str]`: Agrega un hábito nuevo si no existe su clave de identidad. `[RF-2, RF-3]`
  * `mark_habit_done(habits_data: dict, name: str, today: datetime.date) -> tuple[dict, bool, str]`: Registra la fecha de hoy para el hábito. Retorna si fue una nueva marca o si ya estaba marcado. `[RF-5, RF-6, RF-7]`
  * `calculate_streak(completed_dates: set[datetime.date], today: datetime.date) -> int`: Implementa el cálculo de racha actual hacia atrás según D-6. `[RF-9]`
  * `list_habits(habits_data: dict, today: datetime.date) -> list[tuple[str, int]]`: Retorna lista ordenada por punto de código Unicode con `(nombre_mostrado, racha)`. `[RF-8]`
  * `format_streak_text(count: int) -> str`: Concordancia singular/plural (`1 día`, `N días`). `[RF-15]`

#### `habits/storage.py` (Persistencia) `[Cubre RF-11, RF-12, RF-14]`
* **Regla constitucional (P5):** Único módulo autorizado para leer y escribir el archivo JSON.
* **Funciones públicas principales:**
  * `get_storage_path() -> Path`: Resuelve la ruta desde `HABITS_FILE` o el valor por defecto `~/.habits.json`. `[RF-11]`
  * `load_habits(path: Path | None = None, today: datetime.date | None = None) -> dict`: Lee, parsea y valida exhaustivamente el esquema del JSON, fechas ISO y coherencia de fechas futuras contra `today`. Maneja archivos de 0 bytes como almacén vacío. `[RF-11, RF-14, CL-14, CL-15, CL-17]`
  * `save_habits(habits_data: dict, path: Path | None = None) -> None`: Crea directorios padres si no existen y realiza la escritura atómica usando un archivo temporal + `os.replace`. `[RF-11, RF-12, CL-19, CL-25]`
* **Excepciones de persistencia:** `StorageError`, `CorruptDataError`, `InconsistentDateError`, `StoragePermissionError`.

#### `habits/cli.py` (Interfaz CLI) `[Cubre RF-10, RF-13, RF-14, RF-15]`
* **Regla constitucional (P3, P6):** Captura `today` una única vez al inicio, configura `argparse`, invoca a `storage` y `core`, formatea mensajes en español y emite a `stdout` (éxitos) o `stderr` (errores/ayuda) con códigos de salida `0` o `1`/`2`.
* **Funciones principales:**
  * `main(args: list[str] | None = None) -> int`: Orquestador principal que captura excepciones controladas e imprime mensajes limpios sin trazas. `[RF-13, RF-14]`
  * Handlers específicos: `handle_add`, `handle_done`, `handle_list`.

#### `habits/__main__.py`
* Permite la ejecución directa:
  ```python
  import sys
  from habits.cli import main

  if __name__ == "__main__":
      sys.exit(main())
  ```

---

## 2. Modelo de Datos JSON y Esquema

`[Cubre RF-11, RF-14]`

### Esquema Normativo
```json
{
  "habits": {
    "<identity_key>": {
      "name": "<nombre_mostrado>",
      "completed_dates": ["YYYY-MM-DD"]
    }
  }
}
```

### Reglas Estrictas del Modelo de Datos
1. La clave de nivel superior debe ser obligatoriamente `"habits"`.
2. Cada clave dentro de `"habits"` debe ser exactamente igual a `to_identity_key(name)` (`casefold` del nombre normalizado). Si no coincide, se considera dato corrupto.
3. `name`: `str` con longitud entre 1 y 100 caracteres normalizados.
4. `completed_dates`: `list[str]`, donde cada elemento es una cadena con fecha válida de calendario en formato ISO `YYYY-MM-DD`.
5. Fechas posteriores a `today` son rechazadas como incoherencia.
6. Fechas duplicadas o desordenadas en `completed_dates` se procesan como conjunto (`set`), garantizando idempotencia.

### Ejemplo de Archivo JSON (`~/.habits.json`)
```json
{
  "habits": {
    "estudiar python": {
      "name": "Estudiar Python",
      "completed_dates": [
        "2026-08-27",
        "2026-08-28",
        "2026-08-29"
      ]
    },
    "leer libros": {
      "name": "Leer Libros",
      "completed_dates": [
        "2026-08-28"
      ]
    },
    "meditar": {
      "name": "Meditar",
      "completed_dates": []
    }
  }
}
```

---

## 3. Algoritmo de Cálculo de Racha (Pseudocódigo)

`[Cubre D-4, D-5, D-6, RF-9, CL-1..CL-6, CL-11, CL-12, CL-17]`

El cálculo sigue estrictamente la definición D-6: cuenta días consecutivos hacia atrás desde el **día ancla** (*hoy* si hoy está cumplido; o *ayer* si hoy no está cumplido pero ayer sí).

### Pseudocódigo

```text
FUNCTION calculate_streak(completed_dates_set: Set[Date], today: Date) -> Integer:
    IF completed_dates_set IS EMPTY:
        RETURN 0

    // Validar que no existan fechas en el futuro (manejado en validación de carga)
    FOR EACH d IN completed_dates_set:
        IF d > today:
            RAISE InconsistentDateError("Fecha futura detectada")

    yesterday = today - 1 day

    // Determinar el día ancla
    IF today IN completed_dates_set:
        anchor = today
    ELSE IF yesterday IN completed_dates_set:
        anchor = yesterday
    ELSE:
        RETURN 0  // Ni hoy ni ayer están cumplidos; la racha actual es 0

    // Contar días consecutivos hacia atrás desde el ancla
    streak_count = 0
    current_check = anchor

    WHILE current_check IN completed_dates_set:
        streak_count = streak_count + 1
        current_check = current_check - 1 day  // Retroceder un día de calendario

    RETURN streak_count
END FUNCTION
```

### Propiedades del Algoritmo
* **Complejidad Temporal:** $O(K)$, donde $K$ es el número de días de la racha actual activa (búsquedas $O(1)$ en el conjunto `set`).
* **Invarianza al orden y duplicados:** Al convertir `completed_dates` a un conjunto `set[datetime.date]`, las fechas repetidas o desordenadas no afectan el resultado `[CL-12]`.
* **Aritmética de calendario:** El uso de saltos de fecha (`timedelta(days=1)`) preserva correctamente las transiciones de fin de mes y fin de año `[CL-11]`.

---

## 4. Contrato de la CLI

`[Cubre RF-2, RF-5, RF-8, RF-10, RF-13, RF-14, RF-15, CL-20, CL-21]`

### Comandos y Sintaxis

#### 1. `add` — Crear hábito
* **Sintaxis:** `python -m habits add <nombre...>`
* **Argumentos:** Uno o más tokens posicionales (`nargs="+"`), unidos por espacio simple y normalizados.
* **Salida Exitosa (`stdout`, exit code `0`):**
  ```text
  Hábito 'Leer Libros' creado con éxito.
  ```
* **Errores (`stderr`, exit code `1` o `2`):**
  * Sin argumentos: `Error: Se requiere el nombre del hábito.`
  * Nombre vacío/espacios: `Error: El nombre del hábito no puede estar vacío.`
  * Nombre > 100 caracteres: `Error: El nombre del hábito no puede superar los 100 caracteres.`
  * Caracteres de control: `Error: El nombre del hábito contiene caracteres no permitidos.`
  * Hábito duplicado: `Error: Ya existe un hábito con el nombre 'Leer Libros'.`

#### 2. `done` — Marcar cumplido hoy
* **Sintaxis:** `python -m habits done <nombre...>`
* **Argumentos:** Uno o más tokens posicionales (`nargs="+"`).
* **Salida Exitosa (`stdout`, exit code `0`):**
  * Primera marca hoy:
    ```text
    Hábito 'Leer Libros' marcado como cumplido hoy.
    ```
  * Ya estaba marcado hoy:
    ```text
    El hábito 'Leer Libros' ya estaba marcado como cumplido hoy.
    ```
* **Errores (`stderr`, exit code `1` o `2`):**
  * Sin argumentos: `Error: Se requiere el nombre del hábito.`
  * Hábito inexistente: `Error: No existe ningún hábito con el nombre 'NoExiste'.`

#### 3. `list` — Listar hábitos y rachas
* **Sintaxis:** `python -m habits list`
* **Argumentos:** Ninguno. Si se pasan argumentos adicionales, finaliza con error.
* **Salida Exitosa (`stdout`, exit code `0`):**
  * Con hábitos registrados (ordenados por clave de identidad Unicode):
    ```text
    estudiar python: 3 días
    leer libros: 1 día
    meditar: 0 días
    ```
  * Sin hábitos registrados (lista vacía):
    ```text
    No hay hábitos registrados aún. Usa 'add' para crear uno.
    ```
* **Errores (`stderr`, exit code `1` o `2`):**
  * Con argumentos sobrantes: `Error: El comando 'list' no acepta argumentos adicionales.`

#### Invocaciones Globales Inválidas
* **Sin argumentos:** Muestra ayuda concisa en español por `stderr` y sale con código `2`.
* **Comando desconocido:** `Error: Comando 'xyz' no reconocido. Usa 'add', 'done' o 'list'.` (por `stderr`, exit code `2`).
* **Errores de Almacenamiento / Permisos / Corrupción:**
  * JSON corrupto: `Error: El archivo de datos está corrupto o no tiene un formato válido.`
  * Fechas futuras: `Error: Los datos contienen fechas de cumplimiento futuras incompatibles.`
  * Sin permisos: `Error de acceso: No se tienen permisos para leer o escribir los datos.`

### Matriz de Canales y Códigos de Salida

| Escenario | Canal | Código de Salida |
|---|---|---|
| Operación completada con éxito (`add`, `done`, `list`) | `sys.stdout` | `0` |
| `done` cuando ya estaba marcado hoy (informativo) | `sys.stdout` | `0` |
| `list` cuando no hay hábitos | `sys.stdout` | `0` |
| Invocación sin argumentos (ayuda) | `sys.stderr` | `2` |
| Argumentos faltantes, sobrantes o comando desconocido | `sys.stderr` | `2` |
| Error de validación de negocio (duplicado, no existe, > 100 chars) | `sys.stderr` | `1` |
| Error de almacenamiento, permisos o datos corruptos | `sys.stderr` | `1` |

---

## 5. Decisiones Técnicas y Alternativas Descartadas

### D-1: Núcleo 100% Puro con Inyección de Datos y Fecha
* **Decisión:** `core.py` recibe diccionarios puros y la fecha `today: datetime.date` como parámetros explícitos, retornando tuplas o nuevos diccionarios.
* **Justificación:** Garantiza el Principio 3 de la Constitución (núcleo puro sin E/S ni efectos secundarios). Facilita pruebas unitarias instantáneas y deterministas simulando cualquier fecha de hoy o estado de datos sin mocks de sistema.
* **Alternativa descartada:** Clases con métodos que llaman internamente a `datetime.date.today()` o que acceden a disco (ActiveRecord). Descartada por acoplar el dominio al reloj del sistema y al almacenamiento.

### D-2: Persistencia con Escritura Atómica (`os.replace`)
* **Decisión:** Al guardar, `storage.py` escribe en un archivo temporal (`.habits.json.tmp.<pid>`) en el mismo directorio del destino y luego ejecuta `os.replace` hacia el archivo final.
* **Justificación:** Cumple con RF-12 (todo-o-nada) y RNF-6 (integridad). En sistemas POSIX y Windows con Python 3.12+, `os.replace` garantiza que una interrupción o corte eléctrico no deje un JSON truncado a la mitad.
* **Alternativa descartada:** Abrir directamente con `open(path, "w")`. Descartada porque si el proceso muere a mitad del volcado, el archivo original queda vacío o corrupto.

### D-3: Almacenamiento Indexado por Clave `casefold()`
* **Decisión:** El JSON almacena `{"habits": {<identity_key>: {"name": ..., "completed_dates": ...}}}`.
* **Justificación:** Búsqueda, inserción y verificación de duplicados en tiempo $O(1)$. Preserva el nombre mostrado original `name` mientras indexa unívocamente por la clave normalizada insensible a mayúsculas.
* **Alternativa descartada:** Lista de objetos `[{"name": ...}]`. Descartada porque requiere escaneo $O(N)$ en cada operación y permite claves duplicadas si no se controla rigurosamente en cada inserción.

### D-4: Entrada Flexible de Argumentos (`nargs="+"`) en CLI
* **Decisión:** Los comandos `add` y `done` capturan `nargs="+"` y realizan `" ".join(args)` antes de normalizar.
* **Justificación:** Permite al usuario escribir tanto `python -m habits add "leer libros"` como `python -m habits add leer libros` de forma natural en la terminal, colapsando espacios homogéneamente bajo RF-1.
* **Alternativa descartada:** Exigir obligatoriamente comillas (`nargs=1`). Descartada por mala ergonomía de usuario en la terminal.

### D-5: Aislamiento para Pruebas mediante `HABITS_FILE`
* **Decisión:** `storage.get_storage_path()` prioriza la variable de entorno `HABITS_FILE` sobre `~/.habits.json`.
* **Justificación:** Permite que las pruebas automatizadas (unitarias, integración y CLI) apunten a directorios temporales limpios (`tmp_path`) sin tocar jamás los datos reales del usuario.
* **Alternativa descartada:** Modificar el código fuente de `storage.py` durante tests o mockear la función `open()`. Descartada por fragilidad y violación de la separación de entornos.

---

## 6. Estrategia de Pruebas (Test Strategy)

`[Cubre Principio 4 de la Constitución y RF-1 a RF-15]`

La suite de pruebas se organizará en tres niveles de granularidad usando `pytest`:

### 1. Pruebas Unitarias del Núcleo (`tests/test_core.py`)
* **Foco:** Lógica pura en memoria sin E/S.
* **Casos obligatorios por función pública:**
  * `normalize_name`: Espacios sobrantes al inicio/fin, múltiples espacios internos, tabuladores `\t` `[RF-1, CL-7, CL-8]`.
  * `validate_name`: Cadenas vacías, solo espacios, nombres de 100 caracteres exactos (válido), 101 caracteres (inválido), caracteres `\n`, `\r`, `\v`, `\0` `[RF-4, CL-22, CL-23]`.
  * `to_identity_key`: Plegado casefold (`"Leer"` vs `"leer"` igual clave, `"léer"` distinta clave) `[RF-3, CL-9, CL-10]`.
  * `calculate_streak`:
    * Hábito sin marcas $\rightarrow$ racha 0 `[CL-5]`.
    * Creado y marcado hoy $\rightarrow$ racha 1 `[CL-2]`.
    * Marcado ayer pero no hoy $\rightarrow$ racha viva contada hasta ayer `[CL-3]`.
    * Marcado anteayer, ni ayer ni hoy $\rightarrow$ racha 0 `[CL-4]`.
    * Racha rota en el pasado y marcada hoy $\rightarrow$ racha 1 `[CL-6]`.
    * Cruce de fin de mes (ej. 28 Feb a 1 Mar, 31 Dic a 1 Ene) $\rightarrow$ racha continua `[CL-11]`.
    * Fechas desordenadas y duplicadas en el historial $\rightarrow$ racha correcta e idéntica `[CL-12]`.
  * `list_habits`: Ordenación estricta por punto de código Unicode `[RF-8]`.
  * `format_streak_text`: Concordancia `0 días`, `1 día`, `2 días` `[RF-15]`.

### 2. Pruebas de Persistencia y Almacenamiento (`tests/test_storage.py`)
* **Foco:** Lectura, validación de integridad, manejo de errores de SO y atomicidad con `tmp_path`.
* **Casos cubiertos:**
  * Guardar y recargar datos válidos `[RF-11]`.
  * Archivo inexistente $\rightarrow$ retorna estructura vacía sin error `[CL-13]`.
  * Archivo con 0 bytes $\rightarrow$ retorna estructura vacía y permite escritura posterior `[CL-14]`.
  * Archivo con JSON corrupto / sintaxis inválida $\rightarrow$ eleva `CorruptDataError` sin modificar el archivo `[CL-15]`.
  * Fechas con formato inválido (`2026-02-30`, `invalido`) $\rightarrow$ eleva `CorruptDataError` `[CL-24]`.
  * Fechas futuras mayores a `today` $\rightarrow$ eleva `InconsistentDateError` `[CL-17]`.
  * Discrepancia entre clave y nombre (`casefold(name) != key`) $\rightarrow$ eleva `CorruptDataError`.
  * Creación automática de directorios padres faltantes `[CL-25]`.
  * Simulación de escritura atómica (comprobando que el archivo temporal se renombra limpiamente) `[CL-19]`.

### 3. Pruebas de Integración de CLI (`tests/test_cli.py`)
* **Foco:** Experiencia de usuario, `argparse`, códigos de retorno, flujos `stdout`/`stderr` y ausencia total de trazas de excepción.
* **Casos cubiertos:**
  * Ejecutar `python -m habits add` con éxito (salida en `stdout`, exit `0`).
  * Ejecutar `python -m habits add` duplicado (error en `stderr`, exit `1`).
  * Ejecutar `python -m habits done` (primera vez y segunda vez el mismo día con mensaje informativo en `stdout`, exit `0`) `[CL-1]`.
  * Ejecutar `python -m habits done` sobre hábito inexistente (error en `stderr`, exit `1`).
  * Ejecutar `python -m habits list` con lista vacía y con varios hábitos ordenados `[RF-8, RF-10]`.
  * Invocación sin argumentos $\rightarrow$ ayuda en `stderr`, exit `2` `[CL-20]`.
  * Comando desconocido $\rightarrow$ error en `stderr`, exit `2`.
  * Argumentos sobrantes en `list` $\rightarrow$ error en `stderr`, exit `2` `[CL-21]`.
  * Validación de que ante ningún error se imprima un `Traceback` de Python `[RNF-4, RF-13]`.

---

## 7. Matriz de Cobertura de Requisitos (RF)

| Requisito | Módulo Responsable | Prueba Asociada |
|---|---|---|
| **RF-1** (Normalización) | `habits/core.py` | `test_core.py::test_normalize_name_*` |
| **RF-2** (Crear hábito) | `habits/core.py`, `habits/cli.py` | `test_core.py`, `test_cli.py::test_add_success` |
| **RF-3** (Identidad única) | `habits/core.py` | `test_core.py`, `test_cli.py::test_add_duplicate` |
| **RF-4** (Validación nombre) | `habits/core.py` | `test_core.py::test_validate_name_*` |
| **RF-5** (Marcar `done`) | `habits/core.py`, `habits/cli.py` | `test_cli.py::test_done_success` |
| **RF-6** (Marcar 2 veces hoy) | `habits/core.py`, `habits/cli.py` | `test_cli.py::test_done_already_marked` |
| **RF-7** (Hábito inexistente) | `habits/core.py`, `habits/cli.py` | `test_cli.py::test_done_not_found` |
| **RF-8** (Listar y ordenar) | `habits/core.py`, `habits/cli.py` | `test_core.py`, `test_cli.py::test_list_output` |
| **RF-9** (Cálculo de racha) | `habits/core.py` | `test_core.py::test_calculate_streak_*` |
| **RF-10** (Lista vacía) | `habits/cli.py` | `test_cli.py::test_list_empty` |
| **RF-11** (Persistencia y ruta) | `habits/storage.py` | `test_storage.py::test_persistence_*` |
| **RF-12** (Escritura atómica) | `habits/storage.py` | `test_storage.py::test_atomic_write` |
| **RF-13** (Invocación y canales) | `habits/cli.py` | `test_cli.py::test_invocation_and_exit_codes` |
| **RF-14** (Esquema y datos inv.) | `habits/storage.py`, `habits/cli.py` | `test_storage.py::test_corrupt_and_invalid_data` |
| **RF-15** (Forma de mensajes) | `habits/core.py`, `habits/cli.py` | `test_core.py`, `test_cli.py` |
