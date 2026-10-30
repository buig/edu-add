# Bloque 2. Operaciones CRUD, transacciones y buenas prácticas

## 1. Insertar, actualizar y eliminar datos

Estas operaciones también se realizan mediante un **cursor**. Con motores transaccionales como InnoDB (el predeterminado en MySQL) hay que llamar explícitamente a `commit()` tras una secuencia de `INSERT`/`UPDATE`/`DELETE`, ya que Connector/Python desactiva el *autocommit* por defecto.

Encapsulamos estas operaciones en un **repositorio**, una clase cuya única responsabilidad es la persistencia de un tipo de dato concreto:

```python
from dataclasses import dataclass
from datetime import date

@dataclass
class Empleado:
    nombre: str
    apellido: str
    fecha_nacimiento: date
    genero: str              # 'M' o 'F'
    fecha_contratacion: date
    emp_no: int | None = None  # se rellena tras insertar

class RepositorioEmpleados:
    """Operaciones CRUD sobre la tabla employees (y salaries asociada)."""

    def __init__(self, config: dict):
        self.config = config

    def insertar(self, empleado: Empleado) -> Empleado:
        """Inserta un empleado y devuelve el mismo objeto con el emp_no asignado."""
        consulta = (
            "INSERT INTO employees "
            "(first_name, last_name, hire_date, gender, birth_date) "
            "VALUES (%s, %s, %s, %s, %s)"
        )
        datos = (
            empleado.nombre,
            empleado.apellido,
            empleado.fecha_contratacion,
            empleado.genero,
            empleado.fecha_nacimiento,
        )
        with ConexionBD(self.config) as cnx:
            cursor = cnx.cursor()
            cursor.execute(consulta, datos)
            empleado.emp_no = cursor.lastrowid
            cnx.commit()
            cursor.close()
        return empleado

    def insertar_salario(self, emp_no: int, salario: int, desde: date, hasta: date) -> None:
        consulta = (
            "INSERT INTO salaries (emp_no, salary, from_date, to_date) "
            "VALUES (%(emp_no)s, %(salary)s, %(from_date)s, %(to_date)s)"
        )
        datos = {"emp_no": emp_no, "salary": salario, "from_date": desde, "to_date": hasta}
        with ConexionBD(self.config) as cnx:
            cursor = cnx.cursor()
            cursor.execute(consulta, datos)
            cnx.commit()
            cursor.close()

    def actualizar_apellido(self, emp_no: int, nuevo_apellido: str) -> None:
        consulta = "UPDATE employees SET last_name = %s WHERE emp_no = %s"
        with ConexionBD(self.config) as cnx:
            cursor = cnx.cursor()
            cursor.execute(consulta, (nuevo_apellido, emp_no))
            cnx.commit()
            cursor.close()

    def eliminar(self, emp_no: int) -> None:
        consulta = "DELETE FROM employees WHERE emp_no = %s"
        with ConexionBD(self.config) as cnx:
            cursor = cnx.cursor()
            cursor.execute(consulta, (emp_no,))
            cnx.commit()
            cursor.close()

# --- Uso ---
repo = RepositorioEmpleados(DB_CONFIG)

nuevo = Empleado(
    nombre="Geert",
    apellido="Vanderkelen",
    fecha_nacimiento=date(1977, 6, 14),
    genero="M",
    fecha_contratacion=date.today(),
)
nuevo = repo.insertar(nuevo)
repo.insertar_salario(nuevo.emp_no, 50_000, date.today(), date(9999, 1, 1))
```

> ⚠️ **Advertencia — seguridad**: fijaos en que **siempre** pasamos los valores como parámetros (`%s` o `%(nombre)s`), nunca concatenando cadenas (`"...WHERE emp_no = " + str(emp_no)`). Esto es lo que evita los ataques de **inyección SQL**: Connector/Python se encarga de escapar y tipar correctamente cada valor. Más detalles en la sección de [buenas prácticas](#5-buenas-practicas-y-seguridad).

---

## 2. Consulta de datos

Para consultar, también usamos un cursor: tras ejecutar un `SELECT`, el propio cursor es iterable y nos entrega una fila por iteración.

```python
class RepositorioEmpleados(RepositorioEmpleados):   # (continuación)

    def buscar_contratados_entre(self, inicio: date, fin: date) -> list[Empleado]:
        consulta = (
            "SELECT emp_no, first_name, last_name, birth_date, gender, hire_date "
            "FROM employees WHERE hire_date BETWEEN %s AND %s"
        )
        resultado = []
        with ConexionBD(self.config) as cnx:
            cursor = cnx.cursor()
            cursor.execute(consulta, (inicio, fin))
            for emp_no, nombre, apellido, nacimiento, genero, contratacion in cursor:
                resultado.append(
                    Empleado(nombre, apellido, nacimiento, genero, contratacion, emp_no)
                )
            cursor.close()
        return resultado

# --- Uso ---
repo = RepositorioEmpleados(DB_CONFIG)
contratados_1999 = repo.buscar_contratados_entre(date(1999, 1, 1), date(1999, 12, 31))
for emp in contratados_1999:
    print(f"{emp.apellido}, {emp.nombre} — contratado el {emp.fecha_contratacion:%d/%m/%Y}")
```

Connector/Python convierte automáticamente los tipos de MySQL a tipos nativos de Python (`DATE` → `datetime.date`, `DECIMAL` → `decimal.Decimal`, etc.), por lo que podemos formatear `fecha_contratacion` directamente con `{:%d/%m/%Y}` sin conversiones manuales.

> 💡 **Nota**: si la consulta puede devolver muchas filas, considerad usar `cursor = cnx.cursor(buffered=False)` (el valor por defecto) para ir procesando fila a fila sin cargar todo el resultado en memoria de golpe — el mismo principio que vimos en el tema anterior con la lectura de ficheros grandes.

### Cursor con diccionarios

Si preferís trabajar con los resultados como diccionarios en lugar de tuplas posicionales (más legible, menos frágil ante cambios en el `SELECT`), se puede pedir un cursor especial:

```python
cursor = cnx.cursor(dictionary=True)
cursor.execute("SELECT * FROM employees WHERE emp_no = %s", (emp_no,))
fila = cursor.fetchone()
print(fila["first_name"], fila["last_name"])
```

---

## 3. Transacciones

Una **transacción** es un conjunto de sentencias SQL que se ejecutan como si fueran una sola unidad: si cualquiera de ellas falla, ninguna debe quedar aplicada (propiedad de **atomicidad**). Como MySQL (con InnoDB) tiene el *autocommit* desactivado por defecto en Connector/Python, somos nosotros quienes decidimos cuándo confirmar (`commit()`) o deshacer (`rollback()`) los cambios.

Encapsulamos este patrón en un gestor de contexto propio, para que una transacción haga `rollback()` automáticamente si algo falla dentro del bloque:

```python
class Transaccion:
    """Gestor de contexto que hace commit al salir sin errores,
    o rollback si se produce una excepción dentro del bloque with."""

    def __init__(self, cnx: mysql.connector.MySQLConnection):
        self.cnx = cnx

    def __enter__(self):
        return self.cnx.cursor()

    def __exit__(self, exc_type, exc_value, traceback) -> bool:
        if exc_type is None:
            self.cnx.commit()
        else:
            self.cnx.rollback()
        return False   # no silenciamos la excepción, solo hacemos rollback

# --- Uso: mover un empleado de departamento, todo o nada ---
with ConexionBD(DB_CONFIG) as cnx:
    with Transaccion(cnx) as cursor:
        cursor.execute(
            "UPDATE dept_emp SET to_date = %s WHERE emp_no = %s AND dept_no = %s",
            (date.today(), 10001, "d001"),
        )
        cursor.execute(
            "INSERT INTO dept_emp (emp_no, dept_no, from_date, to_date) "
            "VALUES (%s, %s, %s, %s)",
            (10001, "d002", date.today(), date(9999, 1, 1)),
        )
        # si la segunda sentencia lanzase una excepción, la primera
        # también se deshace automáticamente gracias al rollback()
```

---

## 4. Procedimientos y funciones almacenadas

### Procedimientos almacenados

Se ejecutan con `cursor.callproc(nombre, args)`, que acepta una tupla con los argumentos de entrada/salida y devuelve una copia de esa tupla con los valores de salida ya resueltos.

```sql
CREATE PROCEDURE multiply(IN pFac1 INT, IN pFac2 INT, OUT pProd INT)
BEGIN
    SET pProd := pFac1 * pFac2;
END;
```

```python
class RepositorioProcedimientos:
    """Envuelve llamadas a procedimientos almacenados."""

    def __init__(self, config: dict):
        self.config = config

    def multiplicar(self, a: int, b: int) -> int:
        with ConexionBD(self.config) as cnx:
            cursor = cnx.cursor()
            resultado = cursor.callproc("multiply", (a, b, 0))
            cursor.close()
            return resultado[2]   # el tercer argumento es el OUT

# --- Uso ---
rp = RepositorioProcedimientos(DB_CONFIG)
print(rp.multiplicar(5, 6))   # 30
```

Si el procedimiento devuelve conjuntos de resultados (por ejemplo, con un `SELECT` interno), se recuperan con `cursor.stored_results()`:

```python
cursor.callproc("procedimiento_con_select")
for resultado in cursor.stored_results():
    for fila in resultado.fetchall():
        print(fila)
```

### Funciones almacenadas

Las funciones almacenadas devuelven siempre un único valor, así que se ejecutan como una consulta normal. Por ejemplo, la siguiente función calcula la antigüedad de un empleado en años:

```sql
CREATE FUNCTION antiguedad_anios(pEmpNo INT) RETURNS INT
DETERMINISTIC
BEGIN
    DECLARE anios INT;
    SELECT TIMESTAMPDIFF(YEAR, hire_date, CURDATE()) INTO anios
    FROM employees WHERE emp_no = pEmpNo;
    RETURN anios;
END;
```

```python
class RepositorioFunciones:
    """Envuelve llamadas a la función almacenada 'antiguedad_anios'."""

    def __init__(self, config: dict):
        self.config = config

    def antiguedad_anios(self, emp_no: int) -> int:
        with ConexionBD(self.config) as cnx:
            cursor = cnx.cursor()
            cursor.execute("SELECT antiguedad_anios(%s)", (emp_no,))
            valor = cursor.fetchone()[0]
            cursor.close()
            return valor

# --- Uso ---
funciones = RepositorioFunciones(DB_CONFIG)
print(funciones.antiguedad_anios(10001))   # p. ej. 11
```

> `DETERMINISTIC` no es del todo exacto aquí (el resultado cambia cada día, ya que depende de `CURDATE()`), pero evita el error 1418 que da MySQL por defecto al crear funciones en servidores con `log_bin_trust_function_creators` desactivado. Es un atajo muy habitual en entornos de aprendizaje; en un proyecto real se ajustaría esa variable de configuración del servidor en vez de marcar la función como determinista sin serlo del todo.

> Usar **procedimientos y funciones almacenadas** tiene sentido cuando la lógica (p. ej. un cálculo que combina varias tablas) se va a reutilizar desde distintas aplicaciones o lenguajes: al vivir dentro del SGBD, cualquier cliente que se conecte a la base de datos puede usarlos, no solo nuestra aplicación Python.

---

## 5. Buenas prácticas y seguridad

- **Nunca concatenéis valores dentro de una consulta SQL.** Usad siempre marcadores (`%s` o `%(nombre)s`) y pasad los valores como segundo argumento de `execute()`. Es la defensa principal contra la **inyección SQL**.

  ```python
  # ❌ MAL — vulnerable a inyección SQL
  cursor.execute(f"SELECT * FROM employees WHERE last_name = '{apellido}'")

  # ✅ BIEN — el conector escapa y tipa el valor correctamente
  cursor.execute("SELECT * FROM employees WHERE last_name = %s", (apellido,))
  ```

- **Cerrad siempre cursores y conexiones**, preferiblemente con `with` (como en todos los ejemplos de este tema), para no agotar las conexiones disponibles del servidor.
- **Credenciales fuera del código**: variables de entorno o un `.env` no versionado (ver sección 4), nunca contraseñas en claro en el repositorio.
- **Confirmad explícitamente los cambios** (`commit()`) y controlad los errores con `try/except` + `rollback()`, tal como vimos en la sección de transacciones.
- **Usad un *pool* de conexiones** en aplicaciones con muchas peticiones concurrentes (p. ej. un backend web), en lugar de abrir una conexión nueva por cada petición:

  ```python
  from mysql.connector import pooling

  pool = pooling.MySQLConnectionPool(pool_name="mi_pool", pool_size=5, **DB_CONFIG)
  cnx = pool.get_connection()
  ...
  cnx.close()   # devuelve la conexión al pool, no la destruye
  ```

- **Separad la lógica de acceso a datos del resto de la aplicación** usando el patrón *repositorio* que hemos seguido en este tema: si mañana cambiáis de motor de base de datos, o pasáis a usar un ORM como SQLAlchemy, solo tenéis que reescribir la clase repositorio — el resto del programa no debería enterarse.
- Para aplicaciones medianas o grandes, valorad usar un **ORM** (SQLAlchemy, por ejemplo) en lugar de SQL "a mano": genera las consultas por vosotros, gestiona el *pool* de conexiones y reduce el riesgo de errores de tipeo en el SQL. Entender cómo funciona Connector/Python "por debajo", como hemos hecho aquí, ayuda a depurar problemas cuando el ORM no se comporta como esperáis.

---

## 6. Conclusiones

- Un SGBD administra y gestiona la información de una base de datos, ofreciendo distintos niveles de abstracción al programador.
- Cada SGBD tiene su propio protocolo; los **conectores** (como Connector/Python) ofrecen una API común para no depender totalmente de un motor concreto.
- El flujo de trabajo es siempre el mismo: **incorporar el conector → gestionar la conexión → ejecutar sentencias → confirmar o deshacer cambios → cerrar la conexión.**
- Las consultas con parámetros (`%s`) no son solo cuestión de comodidad: son la defensa principal contra la inyección SQL.
- Las transacciones garantizan que un conjunto de sentencias relacionadas se aplique todo o nada.
- Los procedimientos y funciones almacenadas permiten reutilizar lógica de negocio directamente desde el SGBD, independientemente del lenguaje cliente que se conecte.
- Encapsular el acceso a datos en clases (gestores de conexión, repositorios) separa responsabilidades y facilita cambiar de motor de base de datos, o migrar a un ORM, sin reescribir toda la aplicación.

---

## 7. Ejercicios propuestos

1. Amplía `RepositorioEmpleados` con un método `buscar_por_apellido(apellido: str) -> list[Empleado]`.
2. Crea una clase `RepositorioDepartamentos` con métodos `crear`, `listar_todos` y `eliminar`, siguiendo el mismo patrón que `RepositorioEmpleados`.
3. Escribe un procedimiento almacenado `total_empleados_departamento(IN pDeptNo CHAR(4), OUT pTotal INT)` y una función en Python que lo invoque usando `callproc()`.
4. Usando la clase `Transaccion`, implementa un método `despedir_empleado(emp_no)` que, dentro de una única transacción: cierre el registro en `dept_emp` (poniendo `to_date` a hoy), cierre el salario actual en `salaries` y marque al empleado como inactivo. Provoca un error a propósito en uno de los pasos y comprueba que el `rollback()` deshace todo.
5. Investiga la diferencia entre `cursor.fetchone()`, `cursor.fetchmany(n)` y `cursor.fetchall()`. ¿Cuándo usarías cada uno, pensando en el consumo de memoria visto en el tema anterior?
6. (Ampliación) Instala SQLAlchemy y reescribe `RepositorioEmpleados.insertar()` usando el ORM. Compara cuántas líneas de SQL "a mano" te has ahorrado y qué has perdido en control explícito sobre la consulta generada.

!!! tip "Recurso extra: repaso de SQL"
    Si quieres repasar o reforzar los fundamentos de SQL con más calma, **[Hello SQL](https://github.com/mouredev/hello-sql)** es un curso gratuito en español de [mouredev](https://github.com/mouredev): cerca de 80 lecciones (cada una con su código de ejemplo) y unas 7 horas de vídeo, centrado en MySQL (también cubre PostgreSQL).

!!! info "Material de apoyo"
    Junto a los apuntes de esta unidad se entrega un proyecto de ejemplo completo y ejecutable (`schema_employees.sql`, `bbdd.py`, `demo_tema2.py`) que crea la base de datos y recorre, paso a paso, todos los ejemplos de conexión y operaciones CRUD vistos en el tema.

    📦 [Descargar `UP2_files.zip`](../recursos/up2/UP2_files.zip)

---

➡️ **Prácticas de este bloque:** [Práctica 4](../practicas/practica-up2-4.md) · [Práctica 4 Ampliación](../practicas/practica-up2-4-ampliacion.md)
