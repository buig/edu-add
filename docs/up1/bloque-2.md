# Bloque 2. Persistencia en ficheros CSV

## 1. Ficheros binarios

No todos los ficheros pueden procesarse línea a línea. En un fichero binario cada byte tiene un significado concreto y hay que conocer su formato para interpretarlo. Se abre añadiendo `b` al modo (`"rb"`, `"wb"`, `"ab"`).

- `archivo.read(n)` lee `n` bytes.
- `archivo.write(datos)` escribe `bytes` en la posición actual.
- `archivo.tell()` devuelve la posición actual (en bytes, desde el inicio).
- `archivo.seek(desplazamiento, desde)` cambia la posición actual. `desde` puede ser `0` (inicio, por defecto), `1` (posición actual) o `2` (final del fichero).

```python
class GestorBinario:
    """Operaciones básicas sobre un fichero binario."""

    def __init__(self, ruta: str):
        self.ruta = ruta

    def escribir_bytes(self, datos: bytes, modo: str = "wb") -> None:
        with open(self.ruta, modo) as f:
            f.write(datos)

    def leer_bytes(self, n: int = -1) -> bytes:
        """Lee n bytes (o todo el fichero si n es -1)."""
        with open(self.ruta, "rb") as f:
            return f.read(n)

    def tamano(self) -> int:
        with open(self.ruta, "rb") as f:
            f.seek(0, 2)      # nos posicionamos al final
            return f.tell()   # la posición = número de bytes

# --- Uso ---
gestor = GestorBinario("datos.bin")
gestor.escribir_bytes(bytes([72, 111, 108, 97]))   # "Hola" en ASCII
print(gestor.leer_bytes())      # b'Hola'
print(gestor.tamano())          # 4
```

---

## 2. Directorios y rutas con `pathlib`

Tradicionalmente se usaba el módulo `os.path` (`os.path.join("data", "archivo.csv")`), pero en Python moderno se recomienda el módulo **`pathlib`**, más legible y orientado a objetos: una ruta es un objeto, no una simple cadena.

```python
from pathlib import Path

carpeta = Path("data")
carpeta.mkdir(exist_ok=True)        # crea la carpeta si no existe

ruta = carpeta / "archivo.csv"      # el operador / concatena rutas
print(ruta)                         # data/archivo.csv (o data\archivo.csv en Windows)

print(ruta.exists())                # ¿existe el fichero?
print(ruta.suffix)                  # '.csv'
print(ruta.stem)                    # 'archivo'

for fichero in carpeta.glob("*.csv"):
    print(fichero.name)

ruta.write_text("a,b,c\n1,2,3\n", encoding="utf-8")   # crear y escribir directo
print(ruta.read_text(encoding="utf-8"))               # leer directo
```

`Path` funciona igual en Windows, Linux y macOS: gestiona automáticamente el separador de directorios (`/` frente a `\`).

---

## 3. Persistencia de datos

**Persistencia** es la capacidad de guardar el estado de un programa para poder recuperarlo más adelante (lo que el usuario conoce como "Guardar" y "Abrir"). Para un programador implica **serializar** los datos a un fichero (o base de datos) y, en sentido inverso, **deserializarlos** para reconstruirlos en memoria.

Ejemplo que usaremos en toda esta sección: guardar una lista de puntuaciones de un videojuego, cada una con nombre del jugador, puntuación y tiempo empleado.

Para representar cada puntuación usaremos una clase en vez de una tupla suelta, que es más claro y escalable:

```python
from dataclasses import dataclass

@dataclass
class Puntuacion:
    nombre: str
    puntos: int
    tiempo: str
```

`@dataclass` genera automáticamente `__init__`, `__repr__` y `__eq__` a partir de los atributos declarados.

### 3.1 Persistencia en texto plano

```python
class RepositorioPuntuacionesTexto:
    """Guarda y recupera puntuaciones en un fichero de texto separado por comas."""

    def __init__(self, ruta: str):
        self.ruta = ruta

    def guardar(self, puntuaciones: list[Puntuacion]) -> None:
        with open(self.ruta, "w", encoding="utf-8") as f:
            for p in puntuaciones:
                f.write(f"{p.nombre},{p.puntos},{p.tiempo}\n")

    def recuperar(self) -> list[Puntuacion]:
        puntuaciones = []
        with open(self.ruta, "r", encoding="utf-8") as f:
            for linea in f:
                nombre, puntos, tiempo = linea.rstrip("\n").split(",")
                puntuaciones.append(Puntuacion(nombre, int(puntos), tiempo))
        return puntuaciones

# --- Uso ---
repo = RepositorioPuntuacionesTexto("puntuaciones.txt")
repo.guardar([Puntuacion("Pepe", 108, "4:16"), Puntuacion("Juana", 2315, "8:42")])
print(repo.recuperar())
# [Puntuacion(nombre='Pepe', puntos=108, tiempo='4:16'), Puntuacion(nombre='Juana', puntos=2315, tiempo='8:42')]
```

Este enfoque manual funciona, pero es frágil: si un nombre contuviera una coma, el `split(",")` fallaría. Por eso en la práctica se usan formatos ya normalizados como CSV o JSON.

### 3.2 Persistencia en CSV

El módulo `csv` de la librería estándar gestiona correctamente casos como comas dentro de un campo, comillas, etc.

```python
import csv

class RepositorioPuntuacionesCSV:
    """Guarda y recupera puntuaciones usando el módulo csv."""

    CABECERA = ["nombre", "puntos", "tiempo"]

    def __init__(self, ruta: str):
        self.ruta = ruta

    def guardar(self, puntuaciones: list[Puntuacion]) -> None:
        with open(self.ruta, "w", newline="", encoding="utf-8") as f:
            escritor = csv.DictWriter(f, fieldnames=self.CABECERA)
            escritor.writeheader()
            for p in puntuaciones:
                escritor.writerow({"nombre": p.nombre, "puntos": p.puntos, "tiempo": p.tiempo})

    def recuperar(self) -> list[Puntuacion]:
        puntuaciones = []
        with open(self.ruta, "r", newline="", encoding="utf-8") as f:
            lector = csv.DictReader(f)
            for fila in lector:
                puntuaciones.append(Puntuacion(fila["nombre"], int(fila["puntos"]), fila["tiempo"]))
        return puntuaciones

# --- Uso ---
repo = RepositorioPuntuacionesCSV("puntuaciones.csv")
repo.guardar([Puntuacion("Pepe", 108, "4:16"), Puntuacion("Juana", 2315, "8:42")])
print(repo.recuperar())
```

Fichero `puntuaciones.csv` resultante:

```
nombre,puntos,tiempo
Pepe,108,4:16
Juana,2315,8:42
```

> Se usa `DictWriter`/`DictReader` (en vez de `writer`/`reader`) porque trabajar con diccionarios por nombre de campo es más legible y menos propenso a errores que por posición. Nótese también el parámetro `newline=""`, necesario en el módulo `csv` para evitar líneas en blanco adicionales en Windows.

### 3.3 Persistencia en JSON

JSON es hoy el formato más habitual para intercambiar datos entre aplicaciones (APIs, ficheros de configuración, etc.). El módulo `json` convierte directamente entre estructuras Python (listas, diccionarios) y texto JSON.

```python
import json
from dataclasses import asdict

class RepositorioPuntuacionesJSON:
    """Guarda y recupera puntuaciones en formato JSON."""

    def __init__(self, ruta: str):
        self.ruta = ruta

    def guardar(self, puntuaciones: list[Puntuacion]) -> None:
        datos = [asdict(p) for p in puntuaciones]
        with open(self.ruta, "w", encoding="utf-8") as f:
            json.dump(datos, f, indent=2, ensure_ascii=False)

    def recuperar(self) -> list[Puntuacion]:
        with open(self.ruta, "r", encoding="utf-8") as f:
            datos = json.load(f)
        return [Puntuacion(**d) for d in datos]

# --- Uso ---
repo = RepositorioPuntuacionesJSON("puntuaciones.json")
repo.guardar([Puntuacion("Pepe", 108, "4:16"), Puntuacion("Juana", 2315, "8:42")])
print(repo.recuperar())
```

Fichero `puntuaciones.json` resultante:

```json
[
  {
    "nombre": "Pepe",
    "puntos": 108,
    "tiempo": "4:16"
  },
  {
    "nombre": "Juana",
    "puntos": 2315,
    "tiempo": "8:42"
  }
]
```

`asdict()` convierte una `dataclass` en un diccionario; `Puntuacion(**d)` hace lo inverso, "desempaquetando" el diccionario como argumentos con nombre.

### 3.4 Persistencia en binario con `pickle`

Si se decide guardar el estado en binario, Python incluye el módulo `pickle`, que serializa (casi) cualquier objeto Python directamente, sin tener que definir manualmente el formato.

```python
import pickle

class RepositorioPuntuacionesPickle:
    """Guarda y recupera puntuaciones en formato binario pickle."""

    def __init__(self, ruta: str):
        self.ruta = ruta

    def guardar(self, puntuaciones: list[Puntuacion]) -> None:
        with open(self.ruta, "wb") as f:
            pickle.dump(puntuaciones, f)

    def recuperar(self) -> list[Puntuacion]:
        with open(self.ruta, "rb") as f:
            return pickle.load(f)

# --- Uso ---
repo = RepositorioPuntuacionesPickle("puntuaciones.pkl")
repo.guardar([Puntuacion("Pepe", 108, "4:16"), Puntuacion("Juana", 2315, "8:42")])
print(repo.recuperar())
```

El fichero generado no es legible por humanos, sino una secuencia de bytes específica de Python.

> ⚠️ **Importante — seguridad**: `pickle.load()` puede ejecutar código arbitrario si el fichero ha sido manipulado por un tercero. **Nunca** debe usarse `pickle` para cargar ficheros que no sean de plena confianza (por ejemplo, descargados de internet o subidos por un usuario). Para intercambiar datos entre aplicaciones o guardar datos "de cara afuera", usad siempre JSON o CSV; reservad `pickle` para casos internos y de confianza (por ejemplo, guardar un modelo de *machine learning* entrenado).

### Comparativa de formatos

| Formato | Legible por humanos | Portable a otros lenguajes | Uso típico |
|---------|:---:|:---:|---|
| Texto plano manual | ✅ | ✅ | Ejemplos didácticos, ficheros muy sencillos |
| CSV | ✅ | ✅ | Datos tabulares, hojas de cálculo |
| JSON | ✅ | ✅ | APIs, configuración, intercambio general |
| Pickle | ❌ | ❌ (solo Python) | Guardado interno rápido y de confianza |

---

---

➡️ **Práctica de este bloque:** [Práctica 2](../practicas/practica-up1-2.md)
