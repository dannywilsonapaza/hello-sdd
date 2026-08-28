# Constitución — habits-cli

1. **Stack mínimo.** Python 3.12+ y solo biblioteca estándar; la única dependencia de desarrollo es `pytest`. Añadir otra exige modificar antes este archivo.
2. **La spec manda.** Todo cambio de comportamiento se escribe primero en `specs/<feature>/spec.md`; no se acepta código sin spec aprobada que lo respalde.
3. **Núcleo puro.** `habits/core.py` no importa `argparse`, `sys` ni hace E/S ni `print`: recibe datos y devuelve datos. `habits/cli.py` solo parsea argumentos, llama al núcleo e imprime.
4. **Tests siempre verdes.** Cada función pública del núcleo tiene al menos un test de caso normal y uno de borde; `pytest -q` pasa al 100% antes de cada commit.
5. **Persistencia simple y explícita.** Un único archivo JSON local, leído y escrito solo desde `habits/storage.py`; cambiar su formato exige actualizar la spec y documentar la migración.
6. **Idioma fijo.** Identificadores, comentarios y docstrings en inglés; todos los mensajes visibles al usuario, en español.
