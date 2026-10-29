# Práctica 4: Consultas y operaciones CRUD sobre Sakila

⬅️ *Pertenece a la unidad* [UP2 · Bloque 2. Operaciones CRUD, transacciones y buenas prácticas](../up2/bloque-2.md)

!!! info "Material necesario"
    Para esta práctica necesitas la base de datos **Sakila** (esquema + datos) y su diagrama de tablas. Los tienes listos para descargar:

    📦 [Descargar `sakila.zip`](../recursos/up2/sakila.zip) (`sakila-mv-schema.sql` + `sakila-mv-data.sql`)

    📄 [Descargar `sakila.pdf`](../recursos/up2/sakila.pdf) (diagrama de tablas)

## Parte 1

Partiendo de la configuración de la base de datos de Sakila disponible en nuestro entorno de trabajo, realizad la configuración de un nuevo entorno de trabajo.

Para el entorno de trabajo, configurad el entorno virtual:

```bash
mkdir practica4
cd practica4
python3 -m venv .venv
python3 -m pip install mysql-connector
```

## Parte 2

Implementad un menú donde cada opción responda a las siguientes instrucciones sobre los datos almacenados en la base de datos **Sakila**:

### Extracción de datos

1. Obtener el título de cada película y su categoría (relación entre `film`, `film_category` y `category`).
2. Mostrar el nombre completo del cliente y la dirección asociada (`customer` + `address`).
3. Ver las direcciones junto con el nombre de la ciudad (`address` + `city`).
4. Mostrar el título de las películas junto con el idioma en que están disponibles (`film` + `language`).
5. Ver qué tienda tiene cada copia de película (`inventory` + `store`).
6. Obtener los pagos realizados por cada cliente en sus alquileres (`payment`, `rental`, `customer`).

### Nuevos datos

7. Insertamos un nuevo país llamado **"ValenciaLand"**:
    1. Después podemos obtener su `country_id`. Creamos nueva ciudad asociada al país anterior, **"Ciudad del Abanico"**.
    2. Nueva dirección asociada a la ciudad anterior usando el `city_id`:

        ```sql
        ('Calle de los Colibrís, 23', NULL, 'Centro', @city_id, '46000', '600123456')
        ```

### Actualización de datos

8. Actualizar una ciudad existente (por ejemplo, Ciudad del Abanico) y comprobar cómo esa modificación se refleja en las direcciones asociadas a través de `city_id`.

### Borrado de datos

!!! warning "Orden de borrado"
    Ten en cuenta que hay que borrar la información relacionada desde la tabla más externa de la relación hasta la más importante de la relación.

9. Borra un cliente (`id=1`) con actividad. **Tablas implicadas:** `customer`, `payment`, `rental`.
10. Borrar una categoría (`Action`) con películas. **Tablas implicadas:** `category`, `film_category`.

Restaura de nuevo la base de datos.

---

!!! note "Entrega"
    `ApellidosNombre-ADDP4.zip`
