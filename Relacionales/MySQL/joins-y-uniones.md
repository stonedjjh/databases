# Cruces de Tablas y Operaciones de Conjuntos en MySQL (JOINs y UNION)

![Engine](https://img.shields.io/badge/Engine-MySQL%208.0+-4479A1?style=flat&logo=mysql&logoColor=white)
![Focus](https://img.shields.io/badge/Focus-Relational%20Algebra%20%26%20Joins-blue)

Guía técnica y práctica sobre cómo combinar información de múltiples tablas mediante relaciones clave primaria/foránea (JOINs) y cómo consolidar resultados verticales con operadores de conjuntos (UNION / UNION ALL), analizando su fundamento matemático, impacto en memoria y optimizaciones de MySQL 8.0.

---

## Índice de Contenido

- [1. Fundamentos: Álgebra Relacional y el Producto Cartesiano](#1-fundamentos-álgebra-relacional-y-el-producto-cartesiano)
  - [El Peligro del CROSS JOIN Involuntario](#el-peligro-del-cross-join-involuntario)
- [2. INNER JOIN (Intersección Estricta)](#2-inner-join-intersección-estricta)
  - [Sintaxis y Cláusula ON](#sintaxis-y-cláusula-on)
  - [Ejemplo Práctico de Dominio (Concesionario: Vehículos y Concesionarios)](#ejemplo-práctico-de-dominio-concesionario-vehículos-y-concesionarios)
- [3. LEFT JOIN (Preservación de la Tabla Izquierda)](#3-left-join-preservación-de-la-tabla-izquierda)
  - [Mecánica y Relleno con NULLs](#mecánica-y-relleno-con-nulls)
  - [Técnica Profesional: Detección de Registros Huérfanos / Exclusión](#técnica-profesional-detección-de-registros-huérfanos--exclusión)
- [4. RIGHT JOIN y la Convención de la Industria](#4-right-join-y-la-convención-de-la-industria)
  - [Por qué la Industria Estandariza en LEFT JOIN](#por-qué-la-industria-estandariza-en-left-join)
- [5. Algoritmos de Ejecución de JOINs en MySQL 8.0 (Bajo el Capó)](#5-algoritmos-de-ejecución-de-joins-en-mysql-80-bajo-el-capó)
  - [Nested Loop Join (NLJ)](#nested-loop-join-nlj)
  - [La Revolución del Hash Join (MySQL 8.0.20+)](#la-revolución-del-hash-join-mysql-8020)
  - [La Importancia Crítica de Indexar las Foreign Keys](#la-importancia-crítica-de-indexar-las-foreign-keys)
- [6. Operadores de Conjuntos: UNION vs UNION ALL](#6-operadores-de-conjuntos-union-vs-union-all)
  - [Requisitos Estructurales para Unir Resultados](#requisitos-estructurales-para-unir-resultados)
  - [UNION ALL: Concatenación Pura O(N)](#union-all-concatenación-pura-on)
  - [UNION: Concatenación + Deduplicación Costosa](#union-concatenación--deduplicación-costosa)
  - [Regla de Oro en Producción: Evitar UNION Prematuro](#regla-de-oro-en-producción-evitar-union-prematuro)

---

## 1. Fundamentos: Álgebra Relacional y el Producto Cartesiano

En el modelo relacional, los datos se encuentran normalizados en tablas separadas para garantizar consistencia y evitar redundancia. Cuando requerimos relacionar dos entidades, el motor aplica operaciones de álgebra relacional.

La operación más básica entre dos relaciones $A$ y $B$ es el **Producto Cartesiano ($A \times B$)**: cada fila de la primera tabla se empareja indiscriminadamente con todas y cada una de las filas de la segunda tabla.

$$\text{Total de filas resultantes} = |A| \times |B|$$

### El Peligro del CROSS JOIN Involuntario

Si ejecutas un JOIN sin especificar la condición de enlace (`ON`) o usas comas en el `FROM` sin un filtro en el `WHERE`:

```sql
-- ANTIPATRÓN CRÍTICO (CROSS JOIN accidental):
SELECT * 
FROM vehiculos, clientes;
```

> [!CAUTION]
> Si la tabla `vehiculos` tiene **10.000 filas** y la tabla `clientes` tiene **50.000 filas**, esta consulta generará un resultado monstruoso de **500.000.000 de filas** ($10.000 \times 50.000$), saturando instantáneamente la memoria RAM del servidor y provocando un bloqueo de recursos (*Out Of Memory*).

---

## 2. INNER JOIN (Intersección Estricta)

El `INNER JOIN` evalúa la condición especificada en la cláusula `ON` y devuelve **únicamente aquellas tuplas donde exista correspondencia exacta en ambas tablas**:

$$\text{INNER JOIN} = A \cap B$$

Si un registro de la tabla izquierda no tiene contraparte en la tabla derecha (o viceversa), dicho registro queda excluido del resultado final.

### Sintaxis y Cláusula ON

```sql
SELECT 
    t1.columna, 
    t2.columna
FROM tabla_izquierda t1
INNER JOIN tabla_derecha t2 
    ON t1.clave_foranea = t2.clave_primaria;
```

### Ejemplo Práctico de Dominio (Concesionario: Vehículos y Concesionarios)

Imaginemos dos tablas de nuestro dominio automotriz:
- `concesionarios` (`id`, `nombre_comercial`, `ciudad`)
- `vehiculos` (`id`, `vin`, `modelo`, `precio_venta`, `concesionario_id`)

```sql
-- Obtener los vehículos activos y la sede del concesionario donde están asignados
SELECT 
    v.vin,
    v.modelo,
    v.precio_venta,
    c.nombre_comercial AS concesionario,
    c.ciudad
FROM vehiculos v
INNER JOIN concesionarios c 
    ON v.concesionario_id = c.id
WHERE v.estado_publicacion = 'Activo';
```

*(Los vehículos que por alguna razón tengan `concesionario_id IS NULL` o los concesionarios recién abiertos que aún no tengan autos asignados no aparecerán en este reporte).*

---

## 3. LEFT JOIN (Preservación de la Tabla Izquierda)

El `LEFT JOIN` (o `LEFT OUTER JOIN`) devuelve **todas las filas de la tabla izquierda ($A$)**, sin importar si tienen o no coincidencia en la tabla derecha ($B$).

- Si existe correspondencia: Se combinan los atributos de ambas tablas.
- Si no existe correspondencia: Las columnas de la tabla derecha se rellenan con **`NULL`**.

```sql
-- Listar TODOS los concesionarios y los autos que tienen en stock (si los tienen)
SELECT 
    c.nombre_comercial,
    c.ciudad,
    v.vin,
    v.modelo,
    COALESCE(v.precio_venta, 0.00) AS precio_venta
FROM concesionarios c
LEFT JOIN vehiculos v 
    ON c.id = v.concesionario_id;
```

Si una sucursal abrió hoy y no tiene vehículos, aparecerá en el resultado mostrando su nombre y ciudad, pero con `vin = NULL` y `modelo = NULL`.

---

### Técnica Profesional: Detección de Registros Huérfanos / Exclusión

Uno de los usos más potentes del `LEFT JOIN` en auditorías de integridad y migraciones de datos es encontrar registros que **no tienen relación con otra entidad**:

$$\text{Diferencia de Conjuntos} = A - B$$

```sql
-- Caso de Negocio: Encontrar concesionarios fantasmas o sucursales vacías (sin ningún vehículo)
SELECT 
    c.id,
    c.nombre_comercial,
    c.ciudad
FROM concesionarios c
LEFT JOIN vehiculos v 
    ON c.id = v.concesionario_id
WHERE v.id IS NULL; -- Solo conservamos donde la derecha no hizo match
```

> [!TIP]
> **Anti-Join Pattern:** Filtrar con `WHERE tabla_derecha.id IS NULL` después de un `LEFT JOIN` es una de las formas más eficientes en MySQL para identificar registros no asociados o validar integridad referencial previa a crear restricciones de `FOREIGN KEY`.

---

## 4. RIGHT JOIN y la Convención de la Industria

El `RIGHT JOIN` (o `RIGHT OUTER JOIN`) hace exactamente lo inverso al `LEFT JOIN`: preserva **todas las filas de la tabla derecha ($B$)** y rellena con `NULL` si no hay match en la izquierda ($A$).

```sql
-- Espejo del ejemplo anterior
SELECT 
    c.nombre_comercial,
    v.modelo
FROM vehiculos v
RIGHT JOIN concesionarios c 
    ON v.concesionario_id = c.id;
```

### Por qué la Industria Estandariza en LEFT JOIN

En equipos de ingeniería, linters de SQL y guías de estilo profesionales, el uso de `RIGHT JOIN` suele desalentarse o prohibirse.

1. **Flujo de Lectura Humana:** En idiomas occidentales leemos de izquierda a derecha y de arriba a abajo. Un `FROM tabla_principal LEFT JOIN tabla_secundaria` expresa con naturalidad: *"toma mi entidad base y anéxale datos opcionales"*.
2. **Claridad en Encadenamiento Múltiple:** Al encadenar 4 o 5 JOINs consecutivos, mezclar `LEFT JOIN` con `RIGHT JOIN` convierte el plan de ejecución mental en un laberinto ilegible.
3. **Equivalencia Total:** Cualquier consulta escrita con `RIGHT JOIN` puede reescribirse de forma idéntica simplemente invirtiendo el orden de las tablas en el `FROM` usando `LEFT JOIN`.

---

## 5. Algoritmos de Ejecución de JOINs en MySQL 8.0 (Bajo el Capó)

¿Cómo hace físicamente el motor para cruzar dos tablas de 500.000 registros en el disco?

### Nested Loop Join (NLJ)
Es el algoritmo tradicional de bases de datos relacionales:
- El motor lee una fila de la tabla externa (*outer table*).
- Con el valor de la clave de enlace, busca coincidencias en la tabla interna (*inner table*).
- **Si la columna de enlace está indexada:** Cada búsqueda en la tabla interna es un salto binario en el árbol B+ en $O(\log N)$. Es extraordinariamente rápido.
- **Si no hay índice:** Se convertía en un desastre de complejidad cuadrática $O(N \times M)$ recorriendo toda la tabla en cada iteración.

---

### La Revolución del Hash Join (MySQL 8.0.20+)

Históricamente, MySQL era criticado porque carecía del algoritmo **Hash Join** presente en PostgreSQL y Oracle.

A partir de **MySQL 8.0.18** (y generalizado completamente en **8.0.20+** sustituyendo al antiguo *Block Nested Loop*):
1. **Fase de Construcción (Build Phase):** El motor toma la tabla más pequeña en memoria y construye una tabla hash en RAM (`join_buffer_size`) indexando la clave del JOIN.
2. **Fase de Sondeo (Probe Phase):** Recorre la tabla más grande una sola vez en streaming; por cada fila, calcula el hash y busca la coincidencia directa en memoria en tiempo $O(1)$.
3. **Resultado:** Cruces masivos entre tablas no indexadas pasaron de tardar horas a resolverse en segundos.

---

### La Importancia Crítica de Indexar las Foreign Keys

> [!IMPORTANT]
> Aunque el *Hash Join* rescata consultas analíticas mal indexadas, en entornos transaccionales OLTP (APIs web, transacciones bancarias, e-commerce) el estándar de oro sigue siendo **Nested Loop con Índice**.
>
> **Regla Mandatoria:** Toda columna que actúe como Foreign Key (`concesionario_id`, `usuario_id`, `cliente_id`) **debe poseer un índice B+ Tree**.
> - En MySQL, cuando defines formalmente una restricción `FOREIGN KEY`, el motor crea el índice automáticamente si no existe.
> - Si no tienes Foreign Keys explícitas pero enlazas por IDs lógicos, crear el índice manualmente evita que el motor sature memoria o bloquee filas.

---

## 6. Operadores de Conjuntos: UNION vs UNION ALL

A diferencia de los `JOINs` (que combinan tablas **horizontalmente**, agregando columnas hacia los lados), los operadores `UNION` y `UNION ALL` combinan resultados **verticalmente**, apilando filas unas sobre otras.

```text
       JOIN (Horizontal)                  UNION (Vertical)
+---------+     +---------+              +---------+
| Tabla A | <-> | Tabla B |              | Fila 1  | (Consulta A)
+---------+     +---------+              | Fila 2  |
                                         +---------+
                                         | Fila 3  | (Consulta B)
                                         | Fila 4  |
                                         +---------+
```

### Requisitos Estructurales para Unir Resultados
Para que dos o más consultas puedan unirse con `UNION` / `UNION ALL`:
1. Deben proyectar exactamente el **mismo número de columnas**.
2. Las columnas correspondientes en cada posición deben tener **tipos de datos compatibles** (o convertibles implícitamente).
3. Los nombres finales de las columnas del reporte los define la **primera consulta**.

---

### UNION ALL: Concatenación Pura $O(N)$

`UNION ALL` toma todas las filas devueltas por la Consulta 1 y les concatena inmediatamente abajo las filas de la Consulta 2, **sin inspeccionar su contenido ni eliminar duplicados**.

```sql
-- Consolidar un flujo histórico de movimientos vehiculares (entradas de taller + ventas)
SELECT 
    vehiculo_id, 
    fecha_evento, 
    'Mantenimiento en Taller' AS tipo_evento
FROM ordenes_servicio

UNION ALL

SELECT 
    vehiculo_id, 
    fecha_venta AS fecha_evento, 
    'Entrega a Cliente' AS tipo_evento
FROM ventas_completadas

ORDER BY fecha_evento DESC;
```

- **Rendimiento:** Extremadamente rápido. La complejidad es lineal $O(N)$.
- **Consumo de recursos:** Mínimo; MySQL transmite los registros directamente hacia el cliente sin necesidad de retenerlos en buffers temporales.

---

### UNION: Concatenación + Deduplicación Costosa

La sentencia `UNION` a secas concatena los resultados pero ejecuta un paso adicional obligatorio: **elimina todas las filas idénticas duplicadas** (aplica un `DISTINCT` implícito sobre todas las columnas proyectadas).

```sql
-- Obtener todas las ciudades donde tenemos presencia (sea por concesionarios o por clientes registrados)
SELECT ciudad FROM concesionarios
UNION
SELECT ciudad FROM clientes;
```

#### ¿Qué hace el motor tras bambalinas?
1. Ejecuta la Consulta 1.
2. Ejecuta la Consulta 2.
3. Vuelca todos los registros en una **tabla temporal interna** en memoria (o en disco si supera `tmp_table_size`).
4. Ejecuta un algoritmo de ordenamiento/hashing (*filesort*) sobre la combinación de todas las columnas para detectar y purgar duplicados.
5. Devuelve el resultado final limpio.

---

### Regla de Oro en Producción: Evitar UNION Prematuro

| Operador | Deduplica Filas? | Crea Tabla Temporal en RAM/Disco? | Costo Computacional |
| :--- | :---: | :---: | :---: |
| **`UNION ALL`** | **No** (Conserva todas las filas) | **No** (Streaming directo) | $O(N)$ - Ultraligero |
| **`UNION`** | **Sí** (Elimina duplicados) | **Sí** (`Using temporary`) | Alto (Depende de RAM/CPU) |

> [!TIP]
> **Directriz de Arquitectura:**  
> En entornos de producción con alto volumen de datos, utiliza **`UNION ALL` por defecto**.  
> Emplea `UNION` únicamente cuando el requerimiento de negocio exija explícitamente evitar duplicados y no sea viable garantizar la unicidad desde las consultas individuales.
