# Expresiones de Tabla Comunes (CTEs), Funciones de Ventana y Expresiones Regulares en MySQL

![Engine](https://img.shields.io/badge/Engine-MySQL%208.0+-4479A1?style=flat&logo=mysql&logoColor=white)
![Focus](https://img.shields.io/badge/Focus-Advanced%20Analytics%20%26%20Window%20Functions-blue)

Guía técnica y práctica avanzada sobre el diseño de consultas complejas y análisis de datos en MySQL 8.0+. Cubre la modularización con CTEs (`WITH`), el poder analítico de las Funciones de Ventana (`OVER`, `PARTITION BY`, `ROW_NUMBER`, `RANK`, `DENSE_RANK`) sin colapsar registros, y el filtrado avanzado de patrones con `REGEXP`.

---

## Índice de Contenido

- [1. Expresiones de Tabla Comunes (CTEs - Common Table Expressions)](#1-expresiones-de-tabla-comunes-ctes---common-table-expressions)
  - [¿Qué es un CTE y qué problema resuelve?](#qué-es-un-cte-y-qué-problema-resuelve)
  - [Sintaxis con la Cláusula `WITH`](#sintaxis-con-la-cláusula-with)
  - [Múltiples CTEs Encadenados](#múltiples-ctes-encadenados)
  - [CTE vs Subconsultas vs Vistas Temporales](#cte-vs-subconsultas-vs-vistas-temporales)
- [2. Introducción a las Funciones de Ventana (Window Functions)](#2-introducción-a-las-funciones-de-ventana-window-functions)
  - [Diferencia Fundamental: `GROUP BY` vs Funciones de Ventana](#diferencia-fundamental-group-by-vs-funciones-de-ventana)
  - [Anatomía de la Cláusula `OVER()`](#anatomía-de-la-cláusula-over)
- [3. Segmentación con `PARTITION BY` y Ordenamiento Local con `ORDER BY`](#3-segmentación-con-partition-by-y-ordenamiento-local-con-order-by)
  - [Cómo Funciona el Particionado en Memoria](#cómo-funciona-el-particionado-en-memoria)
  - [Interacción entre `PARTITION BY` y `ORDER BY`](#interacción-entre-partition-by-y-order-by)
- [4. Funciones de Clasificación y Ranking](#4-funciones-de-clasificación-y-ranking)
  - [`ROW_NUMBER()`: Numeración Secuencial Única](#row_number-numeración-secuencial-única)
  - [`RANK()`: Posición con Huecos en Empates](#rank-posición-con-huecos-en-empates)
  - [`DENSE_RANK()`: Clasificación Continua sin Huecos](#dense_rank-clasificación-continua-sin-huecos)
  - [Tabla Comparativa Visual de Empates (Tie-Breaking)](#tabla-comparativa-visual-de-empates-tie-breaking)
  - [Patrón de Oro: El Top N por Categoría con CTE + Window Functions](#patrón-de-oro-el-top-n-por-categoría-con-cte--window-functions)
- [5. Búsquedas y Validación Avanzada con `REGEXP` (Expresiones Regulares)](#5-búsquedas-y-validación-avanzada-con-regexp-expresiones-regulares)
  - [Sintaxis y Operadores Principales (`^`, `$`, `[...]`, `|`, `+`, `*`)](#sintaxis-y-operadores-principales------)
  - [Casos Prácticos de Filtrado y Limpieza de Datos](#casos-prácticos-de-filtrado-y-limpieza-de-datos)
  - [Diferencias entre `LIKE` y `REGEXP` (Rendimiento e Índices)](#diferencias-entre-like-y-regexp-rendimiento-e-índices)

---

## 1. Expresiones de Tabla Comunes (CTEs - Common Table Expressions)

### ¿Qué es un CTE y qué problema resuelve?

Un **CTE (Common Table Expression)** es un conjunto de resultados temporal con nombre que existe exclusivamente durante la ejecución de una consulta única (`SELECT`, `INSERT`, `UPDATE` o `DELETE`).

Antes de MySQL 8.0, las consultas complejas requerían anidar múltiples subconsultas en el `FROM` o `WHERE`, provocando el antipatrón conocido como **código spaguetti** o "muñeca rusa", donde la consulta más interna se ejecuta primero y el código se vuelve ilegible y difícil de mantener.

Los CTEs permiten escribir consultas de arriba hacia abajo, de forma modular, lógica y legible.

### Sintaxis con la Cláusula `WITH`

```sql
WITH NombreCTE AS (
    -- Consulta que genera la tabla temporal lógica
    SELECT 
        columna1,
        columna2,
        COUNT(*) AS total
    FROM tabla_origen
    WHERE condicion = 1
    GROUP BY columna1, columna2
)
-- Consulta principal que consume el CTE
SELECT 
    columna1,
    total
FROM NombreCTE
WHERE total > 5;
```

### Múltiples CTEs Encadenados

Puedes definir varios CTEs separándolos por comas, e incluso **hacer que un CTE posterior consuma el resultado de un CTE anterior**:

```sql
WITH ventas_por_cliente AS (
    SELECT 
        customer_id,
        SUM(amount) AS total_gastado,
        COUNT(payment_id) AS total_transacciones
    FROM payment
    GROUP BY customer_id
),
clientes_vip AS (
    SELECT 
        customer_id,
        total_gastado,
        total_transacciones
    FROM ventas_por_cliente
    WHERE total_gastado > 150
)
SELECT 
    c.first_name,
    c.last_name,
    c.email,
    vip.total_gastado,
    vip.total_transacciones
FROM clientes_vip vip
INNER JOIN customer c ON vip.customer_id = c.customer_id
ORDER BY vip.total_gastado DESC;
```

### CTE vs Subconsultas vs Vistas Temporales

| Característica | CTE (`WITH ...`) | Subconsulta derivada (`(SELECT ...) AS sub`) | Vista (`CREATE VIEW`) | Tabla Temporal (`TEMPORARY TABLE`) |
| :--- | :--- | :--- | :--- | :--- |
| **Alcance (Scope)** | Consulta actual | Consulta actual / bloque local | Persistente en catálogo | Sesión de usuario |
| **Legibilidad** | Excelente (Modular y lineal) | Compleja con muchos niveles | Buena | Progresiva en scripts imperativos |
| **Reutilización local** | Puede referenciarse múltiples veces en la misma consulta | Debe reescribirse completa si se reutiliza | Global a todas las consultas | Sesión completa |
| **Sobrecarga de catálogo** | Cero (vive en memoria de parseo) | Cero | Requiere permisos de DDL y metadatos | Reserva espacio temporal en disco/RAM |

---

## 2. Introducción a las Funciones de Ventana (Window Functions)

### Diferencia Fundamental: `GROUP BY` vs Funciones de Ventana

Este es el concepto más transformador de SQL analítico:

- **`GROUP BY` (Colapsa registros):** Toma $N$ filas de entrada y las compacta en una sola fila por cada clave de grupo. Al hacer esto, **pierdes el detalle individual de cada registro**.
- **Funciones de Ventana con `OVER()` (Mantiene los registros):** Calcula un valor agregado o analítico sobre un conjunto de filas relacionadas (la "ventana"), pero **conserva la identidad y el detalle de cada fila individual intacto**.

```
GROUP BY:
[Fila 1 (Dept A: $100)] ──┐
[Fila 2 (Dept A: $200)] ──┴─► [Dept A: $300 (Total)]  <-- Se pierden las filas originales

WINDOW FUNCTION:
[Fila 1 (Dept A: $100)] ────► [Fila 1: Dept A | $100 | Total Dept: $300]
[Fila 2 (Dept A: $200)] ────► [Fila 2: Dept A | $200 | Total Dept: $300]
```

### Anatomía de la Cláusula `OVER()`

La presencia de la palabra clave `OVER()` transforma una función en función de ventana.

```sql
SELECT 
    payment_id,
    customer_id,
    amount,
    -- Calcula el promedio global sin colapsar las filas
    AVG(amount) OVER() AS promedio_global,
    -- Calcula el total acumulado o por cliente según se configure
    SUM(amount) OVER(PARTITION BY customer_id) AS total_cliente
FROM payment;
```

---

## 3. Segmentación con `PARTITION BY` y Ordenamiento Local con `ORDER BY`

### Cómo Funciona el Particionado en Memoria

`PARTITION BY` subdivide el conjunto de datos en subconjuntos o "islas" independientes basadas en los valores de una o más columnas. La función de ventana se reinicia y evalúa por separado para cada partición.

```sql
SELECT 
    film_id,
    title,
    rental_rate,
    rating,
    -- Promedio de precio de alquiler específico para la clasificación (G, PG, R, etc.)
    AVG(rental_rate) OVER(PARTITION BY rating) AS promedio_por_rating,
    -- Diferencia del precio de esta película respecto al promedio de su categoría
    rental_rate - AVG(rental_rate) OVER(PARTITION BY rating) AS dif_vs_promedio
FROM film;
```

### Interacción entre `PARTITION BY` y `ORDER BY`

Dentro de `OVER()`, la cláusula `ORDER BY` define el orden lógico en que se procesan las filas dentro de cada partición:

```sql
SELECT 
    payment_id,
    customer_id,
    payment_date,
    amount,
    -- Suma acumulativa progresiva (Running Total) por cliente
    SUM(amount) OVER(
        PARTITION BY customer_id 
        ORDER BY payment_date
    ) AS total_acumulado_cliente
FROM payment;
```

> [!NOTE]
> Cuando se añade un `ORDER BY` dentro de `OVER()` a funciones de agregación como `SUM()` o `COUNT()`, la ventana por defecto se convierte en acumulativa (`RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`).

---

## 4. Funciones de Clasificación y Ranking

Las funciones de ranking asignan una posición numérica a cada fila dentro de su partición en función del `ORDER BY` especificado.

### `ROW_NUMBER()`: Numeración Secuencial Única

Asigna un entero estrictamente secuencial y consecutivo ($1, 2, 3, 4, \dots$).
- **No admite empates:** Si dos registros tienen el mismo valor de ordenamiento, el motor desempatará de forma no determinista (a menos que añadas criterios secundarios de desempate en el `ORDER BY`).

```sql
SELECT 
    film_id,
    title,
    length,
    ROW_NUMBER() OVER(ORDER BY length DESC) AS fila_num
FROM film;
```

### `RANK()`: Posición con Huecos en Empates

Calcula la posición de una fila dejando **huecos (gaps)** cuando existen empates:
- Si dos filas empatan en el 1.º lugar, ambas reciben el valor `1`.
- La siguiente fila recibirá el valor `3` (salta el `2` porque hay 2 elementos antes).

```sql
SELECT 
    film_id,
    title,
    length,
    RANK() OVER(ORDER BY length DESC) AS posicion_rank
FROM film;
```

### `DENSE_RANK()`: Clasificación Continua sin Huecos

Calcula la posición de una fila **sin saltar ningún número consecutivo** ante empates:
- Si dos filas empatan en el 1.º lugar, ambas reciben el valor `1`.
- La siguiente fila recibirá el valor `2` inmediatamente.

```sql
SELECT 
    film_id,
    title,
    length,
    DENSE_RANK() OVER(ORDER BY length DESC) AS posicion_dense_rank
FROM film;
```

### Tabla Comparativa Visual de Empates (Tie-Breaking)

Supongamos que ordenamos una lista de notas de estudiantes en orden descendente:

| Estudiante | Nota | `ROW_NUMBER()` | `RANK()` | `DENSE_RANK()` | Explicación del Comportamiento |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **Carlos** | 100 | **1** | **1** | **1** | Máxima nota única. |
| **Ana** | 90 | **2** | **2** | **2** | Empate 1 de nota 90. |
| **Beatriz** | 90 | **3** | **2** | **2** | Empate 2 de nota 90. Comparten puesto en `RANK` y `DENSE_RANK`. |
| **David** | 80 | **4** | **4** | **3** | `RANK` salta al 4 (hay 3 estudiantes antes). `DENSE_RANK` sigue en 3. |
| **Elena** | 70 | **5** | **5** | **4** | Siguiente posición correlativa. |

---

### Patrón de Oro: El Top N por Categoría con CTE + Window Functions

> [!IMPORTANT]
> **Regla de Ejecución SQL:** Las funciones de ventana se evalúan en la etapa de proyección (`SELECT`), **después del `WHERE`**. Por lo tanto, no puedes filtrar directamente `WHERE ROW_NUMBER() <= 3` en la misma consulta.
> 
> **Solución estándar:** Envolver la función de ventana dentro de un **CTE** y filtrar en la consulta exterior.

#### Ejemplo: Las 3 películas más largas por cada categoría de clasificación (Rating)

```sql
WITH peliculas_rankeadas AS (
    SELECT 
        film_id,
        title,
        rating,
        length,
        DENSE_RANK() OVER(
            PARTITION BY rating 
            ORDER BY length DESC
        ) AS ranking_duracion
    FROM film
)
SELECT 
    rating,
    ranking_duracion,
    title,
    length
FROM peliculas_rankeadas
WHERE ranking_duracion <= 3
ORDER BY rating, ranking_duracion;
```

Este es uno de los patrones más solicitados en entrevistas técnicas y optimizaciones de reportes de negocio (ej. "Los 5 productos más vendidos por sucursal", "Los 3 mejores salarios por departamento").

---

## 5. Búsquedas y Validación Avanzada con `REGEXP` (Expresiones Regulares)

MySQL soporta coincidencia de patrones por expresiones regulares mediante el operador `REGEXP` (o su sinónimo `RLIKE`).

### Sintaxis y Operadores Principales

```sql
SELECT columnas 
FROM tabla 
WHERE columna REGEXP 'patron';
```

| Operador | Significado | Ejemplo | Coincide con |
| :---: | :--- | :--- | :--- |
| `^` | Inicio estricto de la cadena | `^CAR` | Nombres que empiezan con "CAR" |
| `$` | Fin estricto de la cadena | `ING$` | Palabras que terminan en "ING" |
| `.` | Cualquier carácter único | `C.T` | "CAT", "COT", "CUT" |
| `[abc]` | Cualquier carácter listado dentro de los corchetes | `[AEIOU]` | Contiene al menos una vocal |
| `[^abc]` | Cualquier carácter excepto los listados | `[^0-9]` | Contiene caracteres que no son dígitos |
| `[a-z]` | Rango de caracteres | `[0-9]` | Dígitos del 0 al 9 |
| `a|b` | Operador OR (disyunción) | `ACCION|COMEDIA` | Contiene "ACCION" o "COMEDIA" |
| `*` | Cero o más ocurrencias del elemento precedente | `AB*` | "A", "AB", "ABB", "ABBB" |
| `+` | Una o más ocurrencias del elemento precedente | `AB+` | "AB", "ABB" (requiere al menos una 'B') |
| `{n}` | Exactamente $n$ repeticiones | `[0-9]{4}` | Exactamente 4 dígitos numéricos seguidos |

### Casos Prácticos de Filtrado y Limpieza de Datos

#### 1. Buscar actores cuyo nombre empiece por vocal

```sql
SELECT first_name, last_name 
FROM actor
WHERE first_name REGEXP '^[AEIOU]';
```

#### 2. Encontrar títulos de películas que terminen con un año o número de 4 dígitos

```sql
SELECT title 
FROM film 
WHERE title REGEXP '[0-9]{4}$';
```

#### 3. Validar clientes con estructura de correo electrónico sospechosa o no estándar

```sql
SELECT customer_id, email 
FROM customer 
WHERE email NOT REGEXP '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$';
```

### Diferencias entre `LIKE` y `REGEXP` (Rendimiento e Índices)

| Criterio | `LIKE` (`%`, `_`) | `REGEXP` / `RLIKE` |
| :--- | :--- | :--- |
| **Complejidad de patrones** | Muy básica (solo comodín `%` y único `_`). | Extremadamente potente (clases, rangos, cuantificadores). |
| **Uso de Índices B-Tree** | **SÍ**, únicamente si el patrón es un prefijo fijo (ej. `LIKE 'JUAN%'`). | **NO**. Generalmente fuerza un escaneo completo de tabla (*Full Table Scan*). |
| **Rendimiento** | Muy rápido a nivel de CPU. | Mayor consumo de CPU al evaluar el autómata finito de regex. |
| **Regla de uso** | Úsalo para prefijos simples o filtros cotidianos indexables. | Úsalo para validaciones de formato complejas, limpieza y analítica. |

---

## Resumen de Atajos Mentales

1. **Si necesitas modularizar consultas o evitar subconsultas anidadas:** Usa un **CTE** (`WITH cte_nombre AS (...)`).
2. **Si necesitas agregar o enumerar pero conservar cada fila individual:** Usa **Funciones de Ventana** con `OVER()`.
3. **Si necesitas reiniciar el conteo/cálculo por cada grupo:** Añade `PARTITION BY columna` dentro de `OVER()`.
4. **Si necesitas el Top N por grupo:**
   - Define un CTE con `ROW_NUMBER()`, `RANK()` o `DENSE_RANK()`.
   - Filtra `WHERE posicion <= N` en el `SELECT` exterior.
5. **Si necesitas clasificaciones donde los empates no generen saltos de número:** Usa `DENSE_RANK()`.
6. **Si requieres validar patrones de texto complejos (emails, formatos, dígitos):** Usa `REGEXP`.
