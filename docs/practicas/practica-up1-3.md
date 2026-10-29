# Práctica 3: Agenda de contactos en XML

⬅️ *Pertenece a la unidad* [UP1 · Bloque 3. Persistencia en ficheros XML](../up1/bloque-3.md)

## Parte 1

Partiendo del fichero resultante de la práctica 2, `agenda.csv`, vamos a reescribir el fichero en formato XML usando la librería `ElementTree` de Python.

Una vez transformado el fichero CSV a XML, realizaremos la implementación necesaria para crear un programa que realice las mismas acciones sobre la agenda que el realizado en la práctica 2 sobre el fichero CSV.

El fichero que generará esta primera parte se llamará `agenda.xml`.

!!! note "Sobre `ET.tostring()` en Python 3"
    ```python
    # create a new XML file with the results
    mydata = ET.tostring(data)
    myfile = open("items2.xml", "w")
    myfile.write(mydata)
    ```

    La función `mydata = ET.tostring(data)` devuelve un *bytestring* en Python 3+. Para obtener una cadena de texto plano del XML generado es necesaria la siguiente línea para escribir el XML en fichero:

    ```python
    myfile.write(mydata.decode("utf8"))
    ```

## Parte 2

Por tanto, nuestro programa deberá ser capaz de realizar acciones sobre el fichero XML llamado `agenda.xml`.

A continuación se definen las acciones que tiene que permitir realizar el programa sobre el fichero XML:

1. Iniciar la agenda. Mostrar un menú para usar la agenda.
2. Guardar datos de un contacto.
3. Modificar datos de un contacto.
4. Dar de baja a un contacto (Eliminar).
5. Buscar un contacto.
6. Salir de la agenda (Cerrar).

!!! tip "Estructura XML sugerida"
    ```xml
    <agenda>
        <contactos>
            <contacto id="1">
                <nombre>Antonio</nombre>
                <apellidos>Calabuig</apellidos>
                <email>antonio.calabuig@ieslasenia.org</email>
                <telefono1>961205955</telefono1>
                <telefono2></telefono2>
                <direccion>José Capuz 96, Paiporta</direccion>
            </contacto>
        </contactos>
    </agenda>
    ```

---

!!! note "Entrega"
    `ApellidosNombre-ADDP3.zip`
