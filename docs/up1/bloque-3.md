# Bloque 3. Persistencia en ficheros XML

## 1. Ficheros XML

`ElementTree` (`xml.etree.ElementTree`) es el módulo de la librería estándar para trabajar con XML de forma sencilla.

```python
import xml.etree.ElementTree as ET
```

Fichero de partida, `items.xml`:

```xml
<data>
    <items>
        <item name="item1">item1abc</item>
        <item name="item2">item2abc</item>
    </items>
</data>
```

Encapsulamos las operaciones habituales sobre este XML en una clase:

```python
import xml.etree.ElementTree as ET

class GestorXML:
    """Encapsula operaciones habituales sobre un fichero XML con ElementTree."""

    def __init__(self, ruta: str):
        self.ruta = ruta
        self.tree = None
        self.root = None

    def cargar(self) -> None:
        self.tree = ET.parse(self.ruta)
        self.root = self.tree.getroot()

    def guardar(self, ruta_salida: str | None = None) -> None:
        destino = ruta_salida or self.ruta
        self.tree.write(destino, encoding="unicode")
```

### 1.1 Escritura de XML

Para crear un XML desde cero:

1. Se crea el elemento raíz con `ET.Element(tag)`.
2. Se añaden subelementos con `ET.SubElement(padre, tag, atributos)`.
3. Los atributos también pueden fijarse con `elemento.set(nombre, valor)`.
4. El texto del nodo se asigna con `elemento.text`.

```python
class GestorXML(GestorXML):   # (continuación de la clase anterior)

    @staticmethod
    def crear_ejemplo(ruta_salida: str) -> None:
        data = ET.Element("data")
        items = ET.SubElement(data, "items")

        item1 = ET.SubElement(items, "item")
        item1.set("name", "item1")
        item1.text = "item1abc"

        item2 = ET.SubElement(items, "item")
        item2.set("name", "item2")
        item2.text = "item2abc"

        tree = ET.ElementTree(data)
        ET.indent(tree, space="    ")     # sangría legible (Python 3.9+)
        tree.write(ruta_salida, encoding="unicode")

GestorXML.crear_ejemplo("items2.xml")
```

> `ET.indent()` (disponible desde Python 3.9) añade sangría automáticamente al árbol antes de escribirlo, evitando que todo el XML quede en una sola línea, como ocurría en versiones antiguas.

### 1.2 Lectura de XML

- `find(match)` devuelve el **primer** elemento que coincide.
- `findall(match)` devuelve **todos** los elementos que coinciden, como lista.
- `findtext(match)` devuelve directamente el texto del primer nodo que coincide.

```python
class GestorXML(GestorXML):

    def listar_items(self) -> list[dict]:
        """Devuelve una lista de diccionarios {name, texto} para cada <item>."""
        resultado = []
        for item in self.root.iter("item"):
            resultado.append({"name": item.get("name"), "texto": item.text})
        return resultado

gestor = GestorXML("items.xml")
gestor.cargar()
print(gestor.listar_items())
# [{'name': 'item1', 'texto': 'item1abc'}, {'name': 'item2', 'texto': 'item2abc'}]
```

`root.iter("item")` recorre **todos** los nodos `<item>` del árbol, a cualquier profundidad (a diferencia de `findall`, que por defecto solo busca en los hijos directos salvo que se use la ruta `.//item`).

### 1.3 Modificación de XML

```python
class GestorXML(GestorXML):

    def renombrar_items(self, nuevo_texto: str, nuevo_name: str) -> None:
        for item in self.root.iter("item"):
            item.text = nuevo_texto
            item.set("name", nuevo_name)
            item.set("revisado", "true")   # nuevo atributo

gestor = GestorXML("items.xml")
gestor.cargar()
gestor.renombrar_items("texto nuevo", "item_modificado")
gestor.guardar("newitems.xml")
```

Resultado (`newitems.xml`):

```xml
<data>
    <items>
        <item name="item_modificado" revisado="true">texto nuevo</item>
        <item name="item_modificado" revisado="true">texto nuevo</item>
    </items>
</data>
```

### 1.4 Creación de subelementos

```python
class GestorXML(GestorXML):

    def anadir_seccion(self, tag_seccion: str, tag_hijo: str, texto_hijo: str, atributos: dict) -> None:
        seccion = ET.SubElement(self.root, tag_seccion)
        ET.SubElement(seccion, tag_hijo, atributos).text = texto_hijo

gestor = GestorXML("items.xml")
gestor.cargar()
gestor.anadir_seccion("seconditems", "seconditem", "seconditemabc", {"name2": "secondname2"})
gestor.guardar("newitems2.xml")
```

Resultado:

```xml
<data>
    <items>
        <item name="item1">item1abc</item>
        <item name="item2">item2abc</item>
    </items>
    <seconditems>
        <seconditem name2="secondname2">seconditemabc</seconditem>
    </seconditems>
</data>
```

### 1.5 Borrado de elementos

```python
class GestorXML(GestorXML):

    def eliminar_atributo(self, tag: str, atributo: str) -> None:
        for elem in self.root.iter(tag):
            elem.attrib.pop(atributo, None)

    def eliminar_primer_hijo(self, tag_padre: str) -> None:
        padre = self.root.find(tag_padre)
        if padre is not None and len(padre):
            padre.remove(padre[0])

    def vaciar(self, tag_padre: str) -> None:
        padre = self.root.find(tag_padre)
        if padre is not None:
            padre.clear()
```

```python
gestor = GestorXML("items.xml")
gestor.cargar()
gestor.eliminar_atributo("item", "name")     # quita el atributo "name" de todos los <item>
gestor.guardar("newitems3.xml")

gestor2 = GestorXML("items.xml")
gestor2.cargar()
gestor2.eliminar_primer_hijo("items")         # borra el primer <item>
gestor2.guardar("newitems4.xml")

gestor3 = GestorXML("items.xml")
gestor3.cargar()
gestor3.vaciar("items")                       # borra TODOS los <item>
gestor3.guardar("newitems5.xml")
```

---

## 2. Buenas prácticas y errores comunes

- Usad siempre `with open(...) as f:` en lugar de `open()` + `close()` manual.
- Indicad siempre `encoding="utf-8"` al trabajar con texto, para evitar problemas de tildes/`ñ` entre sistemas operativos.
- No uséis `"w"` cuando lo que queréis es añadir contenido: sobrescribe el fichero entero. Usad `"a"`.
- Capturad los errores esperables con `try/except` en vez de dar por hecho que el fichero existe:

```python
try:
    with open("configuracion.json", "r", encoding="utf-8") as f:
        datos = json.load(f)
except FileNotFoundError:
    print("El fichero de configuración no existe, se usarán valores por defecto")
except json.JSONDecodeError:
    print("El fichero de configuración tiene un formato inválido")
```

- Para rutas de fichero, preferid `pathlib.Path` a concatenar cadenas con `+` o `os.path.join`.
- No uséis `pickle` para leer ficheros que no sean de plena confianza (riesgo de seguridad).
- Evitad `read()`/`readlines()` sobre ficheros muy grandes; procesad línea a línea.

---

## 3. Conclusiones

- Para usar un fichero desde un programa hay que abrirlo y, cuando ya no se necesite, cerrarlo (idealmente con `with`).
- Las operaciones básicas sobre un fichero son leer y escribir.
- Todo fichero abierto tiene asociada una posición de lectura/escritura, consultable con `tell()` y modificable con `seek()`.
- Los ficheros de texto se procesan normalmente línea a línea y sirven para intercambiar información entre programas, o entre un programa y una persona.
- Cada formato de fichero binario tiene sus propias reglas.
- Se puede acceder a los datos de forma secuencial, o saltando a posiciones concretas del fichero, según convenga.
- Cargar el contenido completo de un fichero en memoria puede ser innecesario y costoso; hay que evaluarlo según el tamaño del fichero.
- Para datos estructurados, es preferible usar un formato estándar (CSV, JSON, XML) y su módulo correspondiente antes que "reinventar" un formato propio con `split()`/`join()` manuales.
- Encapsular el acceso a ficheros en clases (un "repositorio" o "gestor") separa la lógica de persistencia del resto de la aplicación y facilita cambiar de formato en el futuro sin tocar el resto del código.

---

## 4. Ejercicios propuestos

1. Amplía `RepositorioPuntuacionesJSON` para que ordene las puntuaciones de mayor a menor antes de guardarlas.
2. Crea una clase `GestorAgenda` que guarde contactos (nombre, teléfono, email) en un CSV, con métodos `anadir_contacto`, `buscar_por_nombre` y `eliminar_contacto`.
3. Modifica `Logger` para que además de escribir en el fichero, muestre por consola los mensajes de nivel `ERROR`.
4. Escribe una clase `GestorXML` (partiendo de la vista en el tema) que permita convertir el `items.xml` de ejemplo a una lista de diccionarios y, a partir de ella, generar un fichero `items.json` equivalente usando el módulo `json`.
5. Investiga y explica con tus palabras por qué `pickle.load()` se considera potencialmente inseguro y qué alternativa usarías para compartir datos entre dos aplicaciones distintas.

---

➡️ **Prácticas de este bloque:** [Práctica 3](../practicas/practica-up1-3.md) · [Práctica 3 Ampliación](../practicas/practica-up1-3-ampliacion.md)
