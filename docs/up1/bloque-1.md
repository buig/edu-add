# Bloque 1. Acceso a ficheros

## 1. Tipos de ficheros

Desde el punto de vista de un programador distinguimos dos grandes tipos de ficheros:

- **Ficheros de texto**: su contenido son exclusivamente caracteres de texto (se pueden leer con un editor de texto plano).
- **Ficheros binarios**: no están compuestos exclusivamente de texto. Pueden contener imágenes, vídeo, audio, ejecutables... Incluso un `.docx` es en realidad un fichero binario (un ZIP con XML dentro), aunque "solo" contenga texto visible para el usuario.

### Extensiones habituales de ficheros de texto

| Extensión | Tipo de fichero                     |
|-----------|--------------------------------------|
| `.txt`    | Texto plano                          |
| `.json`   | Fichero de intercambio de información (JSON) |
| `.xml`    | Fichero XML                          |
| `.csv`    | Valores separados por comas          |
| `.cfg` / `.conf` / `.ini` | Fichero de configuración |
| `.sql`    | Script SQL                           |
| `.log`    | Fichero de registro (log)            |
| `.py`     | Código fuente Python                 |

### Tipos de ficheros de texto que vamos a trabajar

- **Texto plano**: contenido libre, sin formato obligatorio.
- **Ficheros de configuración**: información de configuración de una aplicación, con un formato concreto (clave-valor, INI, `.env`...).
- **Ficheros estructurados** (XML, JSON, CSV): contienen datos acompañados de una sintaxis que les da significado. Python incluye módulos de la librería estándar para trabajar cómodamente con todos ellos: `xml.etree.ElementTree`, `json` y `csv`.

## 2. Apertura y cierre de ficheros

Existen dos formas básicas de acceder a un fichero:

- Como **fichero de texto**, procesándolo línea a línea.
- Como **fichero binario**, procesándolo byte a byte.

En Python se abre un fichero con la función `open()`:

```python
archivo = open("archivo.txt")   # modo lectura por defecto ("r")
```

`open()` devuelve un objeto de tipo fichero que permite manipularlo. Este objeto mantiene internamente una **posición de lectura/escritura**, que avanza a medida que se lee o escribe.

### Lectura línea a línea

```python
linea = archivo.readline()
while linea != "":
    print(linea, end="")
    linea = archivo.readline()
```

Forma equivalente, más "pythónica" (el objeto fichero es iterable):

```python
for linea in archivo:
    print(linea, end="")
```

### Leer todo el fichero de una vez

```python
lineas = archivo.readlines()   # lista de cadenas, una por línea
contenido = archivo.read()     # todo el contenido como una única cadena
```

> ⚠️ **Advertencia**: `readlines()` y `read()` cargan el fichero **completo** en memoria. Solo deben usarse con ficheros de tamaño razonable; para ficheros muy grandes es mejor procesar línea a línea.

### Cierre del fichero

```python
archivo.close()
```

Es importante cerrar siempre un fichero al terminar de usarlo:

- En algunos sistemas un fichero solo puede estar abierto por un programa a la vez.
- Lo escrito puede no guardarse realmente en disco (queda en el buffer) hasta que se cierra.
- El número de ficheros que un proceso puede tener abiertos simultáneamente es limitado.

En la sección 4 veremos cómo evitar tener que acordarnos de cerrar el fichero manualmente.

### Ejemplo: numerar las líneas de un fichero

```python
# numera_lineas.py: imprime las líneas de un archivo con su número
archivo = open("archivo.txt")
for i, linea in enumerate(archivo, start=1):
    linea = linea.rstrip("\n")
    print(f"{i:4d}: {linea}")
archivo.close()
```

`enumerate()` genera automáticamente un contador para cada elemento recorrido (aquí, cada línea). `rstrip("\n")` elimina el salto de línea final, ya que cada línea leída lo incluye.

> **Nota sobre finales de línea**: existen tres convenios distintos para marcar el fin de línea: `\n` (Unix/Linux/macOS moderno), `\r` (Mac clásico) y `\r\n` (Windows). Python gestiona esto automáticamente mediante el modo *universal newlines*, activado por defecto: siempre verás `\n`, independientemente del sistema donde se creara el fichero.

## 3. Modos de apertura

`open()` recibe un segundo parámetro opcional que indica el modo de apertura:

| Modo | Significado |
|------|-------------|
| `r`  | Solo lectura. El fichero debe existir, si no, se lanza `FileNotFoundError`. |
| `w`  | Solo escritura. Si el fichero existe, se **trunca** (se vacía); si no existe, se crea. |
| `a`  | Escritura al final (*append*). Si existe, se mantiene el contenido y se escribe a partir del final; si no existe, se crea. |
| `x`  | Creación exclusiva. Crea el fichero, pero falla con `FileExistsError` si ya existe. |
| `r+` | Lectura y escritura. El fichero debe existir y no se trunca. |
| `w+` | Lectura y escritura, truncando el fichero si existe. |
| `b`  | Se combina con los anteriores para modo binario (`rb`, `wb`, `ab`...). |
| `t`  | Modo texto (por defecto, no hace falta indicarlo: `rt` ≡ `r`). |

```python
archivo = open("datos.txt", "r")    # solo lectura
archivo = open("datos.txt", "w")    # solo escritura, trunca
archivo = open("datos.txt", "a")    # añadir al final
archivo = open("datos.bin", "rb")   # lectura binaria
```

> ⚠️ **Advertencia**: abrir un fichero existente en modo `w` o `w+` **borra todo su contenido anterior**. Es un error muy habitual del alumnado perder datos por usar `"w"` en vez de `"a"`.

---

## 4. La sentencia `with`

La forma recomendada (y la que debéis usar siempre en vuestro código) de trabajar con ficheros es mediante un **gestor de contexto** (*context manager*), con la sentencia `with`. Se encarga de cerrar el fichero automáticamente, incluso si se produce una excepción dentro del bloque:

```python
with open("archivo.txt", "w") as archivo:
    archivo.write("Texto agregado\n")
# al salir del bloque, el fichero ya está cerrado, aunque haya habido un error
```

```python
with open("hola.txt") as archivo:
    print(archivo.readlines())
# no hace falta llamar a archivo.close()
print("Ya hemos terminado con el archivo")
```

Se pueden abrir varios ficheros a la vez en el mismo `with`:

```python
with open("entrada.txt") as f_in, open("salida.txt", "w") as f_out:
    for linea in f_in:
        f_out.write(linea.upper())
```

A partir de aquí, **todos los ejemplos del tema usarán `with`**.

---

## 5. Lectura y escritura de ficheros de texto (POO)

En lugar de repetir código de apertura/lectura/escritura cada vez, encapsulamos esa lógica en una clase. Esto es exactamente el enfoque que usaréis en el resto del ciclo: separar la lógica de acceso a datos del resto del programa.

```python
class GestorTexto:
    """Encapsula operaciones básicas sobre un fichero de texto plano."""

    def __init__(self, ruta: str):
        self.ruta = ruta

    def leer_lineas(self) -> list[str]:
        """Devuelve las líneas del fichero, sin el salto de línea final."""
        with open(self.ruta, "r", encoding="utf-8") as f:
            return [linea.rstrip("\n") for linea in f]

    def escribir(self, texto: str, modo: str = "w") -> None:
        """Escribe texto en el fichero. modo='w' sobrescribe, 'a' añade."""
        with open(self.ruta, modo, encoding="utf-8") as f:
            f.write(texto)

    def escribir_lineas(self, lineas: list[str], modo: str = "w") -> None:
        """Escribe una lista de líneas, añadiendo '\\n' a cada una."""
        with open(self.ruta, modo, encoding="utf-8") as f:
            f.writelines(linea + "\n" for linea in lineas)

    def numerar_lineas(self) -> None:
        """Imprime por pantalla cada línea precedida de su número."""
        for i, linea in enumerate(self.leer_lineas(), start=1):
            print(f"{i:4d}: {linea}")

# --- Uso ---
gestor = GestorTexto("saludo.txt")
gestor.escribir("Hola mundo\nSegunda línea\n")
gestor.numerar_lineas()
gestor.escribir("Otra línea más\n", modo="a")
```

> 💡 Nota didáctica: especificamos siempre `encoding="utf-8"` de forma explícita. Si no se indica, Python usa la codificación por defecto del sistema operativo, que puede variar entre Windows, Linux y macOS y provocar errores con tildes o la letra `ñ`.

---

## 6. Ficheros de log

Un uso muy habitual de la apertura en modo `a` (*append*) es el registro de eventos de una aplicación (*logging*), muy usado por administradores de sistemas para depurar problemas.

```python
from datetime import datetime

class Logger:
    """Gestiona un fichero de log sencillo."""

    def __init__(self, ruta: str):
        self.ruta = ruta
        self._registrar("Iniciando registro de eventos")

    def _registrar(self, mensaje: str) -> None:
        hora_actual = datetime.now().isoformat(sep=" ", timespec="seconds")
        with open(self.ruta, "a", encoding="utf-8") as f:
            f.write(f"{hora_actual} {mensaje}\n")

    def info(self, mensaje: str) -> None:
        self._registrar(f"INFO: {mensaje}")

    def error(self, mensaje: str) -> None:
        self._registrar(f"ERROR: {mensaje}")

    def cerrar(self) -> None:
        self._registrar("Fin del registro de eventos")

# --- Uso ---
log = Logger("mi_log.txt")
log.info("Aplicación arrancada correctamente")
try:
    1 / 0
except ZeroDivisionError as e:
    log.error(f"división por cero: {e}")
log.cerrar()
```

Ejemplo de fichero generado:

```
2026-09-10 09:15:32 INFO: Iniciando registro de eventos
2026-09-10 09:15:32 INFO: Aplicación arrancada correctamente
2026-09-10 09:15:33 ERROR: división por cero: division by zero
2026-09-10 09:20:01 INFO: Fin del registro de eventos
```

> Python incluye además el módulo `logging` de la librería estándar, mucho más completo (niveles de log, rotación de ficheros, formato configurable...). Este ejemplo manual sirve para entender el concepto antes de usar la herramienta profesional.

---

---

➡️ **Práctica de este bloque:** [Práctica 1](../practicas/practica-up1-1.md)
