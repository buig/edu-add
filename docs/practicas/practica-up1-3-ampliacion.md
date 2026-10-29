# Práctica 3 Ampliación: Agenda en XML con atributos

⬅️ *Pertenece a la unidad* [UP1 · Bloque 3. Persistencia en ficheros XML](../up1/bloque-3.md)

## Parte 1

Reescribe el fichero creado en la práctica 3 en formato XML, para que todos los datos de un contacto estén como **atributos**.

Para ello puedes partir del fichero en formato `.csv` creado en la práctica 2.

Adapta las funciones de la agenda para que sean capaces de trabajar con el XML en formato de atributos.

!!! tip "Estructura XML sugerida"
    ```xml
    <agenda>
        <contactos>
            <contacto id="1" nombre="Antonio" apellidos="Calabuig"
                email="antonio.calabuig@ieslasenia.org"
                telefono1="961205955" telefono2="" direccion="José Capuz 96, Paiporta" />
            ...
        </contactos>
    </agenda>
    ```

---

!!! note "Entrega"
    `ApellidosNombre-ADDP3.zip`
