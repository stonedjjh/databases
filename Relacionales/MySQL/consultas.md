# Consultas en MySQL (Queries)

![Engine](https://img.shields.io/badge/Engine-MySQL%208.0+-4479A1?style=flat&logo=mysql&logoColor=white)
![Focus](https://img.shields.io/badge/Focus-DML%20%26%20Retrieval-blue)

Guía práctica sobre el diseño, ejecución y optimización de consultas para la extracción de datos en MySQL/InnoDB.

---

## ¿Qué es una Consulta (Query)?

Una consulta (query) es una instrucción o solicitud formal que se envía al motor de base de datos para recuperar, filtrar, ordenar o transformar información almacenada en una o varias tablas. 

En bases de datos relacionales, las consultas de lectura se construyen mediante el lenguaje declarativo SQL: el usuario especifica **qué** datos necesita y bajo qué condiciones, mientras que el optimizador del motor decide **cómo** acceder físicamente a los registros (a través de índices o lecturas secuenciales de disco).

> [!NOTE]
> Las bases de datos relacionales están fuertemente ligadas a la teoría de conjuntos y al álgebra relacional (proyección $\pi$ y restricción/selección $\sigma$). A lo largo de esta guía conectamos estos conceptos para entender exactamente qué hace el motor tras bambalinas.

---

## Índice de Contenido

- [1. Cláusula SELECT (Proyección)](#1-cláusula-select-proyección)
  - [Sintaxis Básica](#sintaxis-básica)
  - [Ejemplos de Uso](#ejemplos-de-uso)
  - [Buenas Prácticas](#buenas-prácticas)
- [2. Cláusula WHERE (Filtrado y Restricción)](#2-cláusula-where-filtrado-y-restricción)
  - [Sintaxis y Orden Lógico de Ejecución](#sintaxis-y-orden-lógico-de-ejecución)
  - [Operadores de Comparación y Lógicos](#operadores-de-comparación-y-lógicos)
  - [La Trampa de los NULL: Lógica Trivalente (Three-Valued Logic)](#la-trampa-de-los-null-lógica-trivalente-three-valued-logic)
  - [Bajo el Capó: Sargabilidad y Uso de Índices](#bajo-el-capó-sargabilidad-y-uso-de-índices)
  - [Buenas Prácticas en el WHERE](#buenas-prácticas-en-el-where)
- [3. Cláusula DISTINCT](#3-cláusula-distinct)
  - [Sintaxis y Ejemplos](#sintaxis-y-ejemplos)
  - [Perspectiva Matemática y de Programación (Set vs Multiset)](#perspectiva-matemática-y-de-programación-set-vs-multiset)
  - [Impacto en Rendimiento e Índices (Loose Index Scan vs Temporary Table)](#impacto-en-rendimiento-e-índices-loose-index-scan-vs-temporary-table)
  - [Criterio de Arquitectura (BD vs Backend)](#criterio-de-arquitectura-bd-vs-backend)
- [4. Cláusula ORDER BY](#4-cláusula-order-by)
  - [Sintaxis y Dirección (ASC vs DESC)](#sintaxis-y-dirección-asc-vs-desc)
  - [Ordenamiento Múltiple y Manejo de NULLs](#ordenamiento-múltiple-y-manejo-de-nulls)
  - [Bajo el Capó: Cómo Ordena el Motor (Index Scan vs Filesort)](#bajo-el-capó-cómo-ordena-el-motor-index-scan-vs-filesort)
- [5. Cláusula LIMIT y Paginación](#5-cláusula-limit-y-paginación)
  - [Sintaxis y Uso Básico](#sintaxis-y-uso-básico)
  - [Paginación Tradicional con OFFSET](#paginación-tradicional-con-offset)
  - [El Problema de Rendimiento: Paginación Profunda (Deep Paging)](#el-problema-de-rendimiento-paginación-profunda-deep-paging)
  - [Solución Profesional: Paginación por Cursor (Keyset Pagination)](#solución-profesional-paginación-por-cursor-keyset-pagination)
- [6. Funciones de Agregación](#6-funciones-de-agregación)
  - [Concepto y Propósito en SQL](#concepto-y-propósito-en-sql)
  - [El Catálogo Esencial: COUNT, SUM, AVG, MIN, MAX](#el-catálogo-esencial-count-sum-avg-min-max)
  - [COUNT(*) vs COUNT(columna): El Manejo de NULLs](#count-vs-countcolumna-el-manejo-de-nulls)
  - [Uso de Alias Semánticos (AS)](#uso-de-alias-semánticos-as)
  - [Filtrado Previo con WHERE](#filtrado-previo-con-where)
  - [Impacto en Rendimiento e Índices B+ Tree](#impacto-en-rendimiento-e-índices-b-tree)
- [7. Cláusula GROUP BY (Agrupamiento de Datos)](#7-cláusula-group-by-agrupamiento-de-datos)
  - [Concepto y Sintaxis Básica](#concepto-y-sintaxis-básica)
  - [La Regla de Oro y ONLY_FULL_GROUP_BY en MySQL 8.0](#la-regla-de-oro-y-only_full_group_by-en-mysql-80)
  - [Agrupación por Múltiples Columnas](#agrupación-por-múltiples-columnas)
  - [Manejo de NULLs en el Agrupamiento](#manejo-de-nulls-en-el-agrupamiento)
  - [Bajo el Capó: Tablas Temporales vs Loose Index Scan](#bajo-el-capó-tablas-temporales-vs-loose-index-scan)
- [8. Cláusula HAVING (Filtrado de Grupos Agregados)](#8-cláusula-having-filtrado-de-grupos-agregados)
  - [Diferencia Fundamental: WHERE vs HAVING](#diferencia-fundamental-where-vs-having)
  - [Sintaxis y Ejemplos de Negocio](#sintaxis-y-ejemplos-de-negocio)
  - [Buenas Prácticas y Rendimiento: Filtrar Temprano (Pushdown)](#buenas-prácticas-y-rendimiento-filtrar-temprano-pushdown)

---

## 1. Cláusula SELECT (Proyección)

La sentencia `SELECT` es el comando fundamental de SQL utilizado para leer y proyectar atributos desde la base de datos.

### Sintaxis Básica

```sql
SELECT columna1, columna2, ...
FROM nombre_tabla;
```

### Ejemplos de Uso

1. **Consultar todas las columnas de una tabla:**
   ```sql
   SELECT * FROM usuarios;
   ```

2. **Proyección explícita de columnas:**
   ```sql
   SELECT id, nombre, email FROM usuarios;
   ```

3. **Uso de alias para renombrar columnas en la salida:**
   ```sql
   SELECT nombre AS nombre_completo, email AS correo_contacto 
   FROM usuarios;
   ```

### Buenas Prácticas
- **Sensibilidad de nombres según el Sistema Operativo (Windows vs Linux):**  
  Aunque las palabras clave son insensibles, en MySQL los nombres de tablas y bases de datos corresponden a carpetas y archivos `.ibd` en disco. Esta sensibilidad depende de `@@lower_case_table_names` (en Windows es `1` y convierte a minúsculas; en Linux es `0` y distingue mayúsculas). Para evitar fallos en despliegues, usa siempre **minúsculas con snake_case** (`usuarios_activos`).
- **Evitar `SELECT *` en entornos de producción y APIs:** Solicitar solo los campos requeridos reduce el tráfico de red, disminuye el consumo de memoria del *Buffer Pool* y permite al optimizador usar índices de cobertura (*covering index*).

---

## 2. Cláusula WHERE (Filtrado y Restricción)

En álgebra relacional, el `WHERE` representa la **operación de selección/restricción ($\sigma$)**: filtra las tuplas (filas) de una tabla evaluando un predicado booleano, permitiendo que solo las filas donde la condición sea **verdadera (`TRUE`)** pasen a la siguiente etapa.

### Sintaxis y Orden Lógico de Ejecución

```sql
SELECT columna1, columna2
FROM nombre_tabla
WHERE condicion_predicado;
```

> [!IMPORTANT]
> ### Orden Sintáctico vs Orden Lógico de Ejecución
> Aunque escribes `SELECT` antes que `WHERE`, el motor ejecuta las cláusulas en este orden lógico:
> 1. **`FROM`**: Identifica las tablas y realiza los JOINs.
> 2. **`WHERE`**: Filtra las filas individuales **antes** de agrupar o proyectar.
> 3. **`GROUP BY`**: Agrupa los registros restantes.
> 4. **`HAVING`**: Filtra grupos agregados.
> 5. **`SELECT`**: Proyecta las columnas y evalúa expresiones/alias.
> 6. **`DISTINCT`**: Deduplica filas si se especificó.
> 7. **`ORDER BY`**: Ordena el resultado final.
> 8. **`LIMIT`**: Corta el número de filas a devolver.
>
> **Consecuencia directa:** ¡No puedes usar alias de columnas creados en el `SELECT` dentro del `WHERE` porque el `WHERE` se ejecuta antes!

```sql
-- ESTO FALLA con: ERROR 1054 (Unknown column 'precio_con_iva')
SELECT nombre, (precio * 1.19) AS precio_con_iva
FROM productos
WHERE precio_con_iva > 100;

-- FORMA CORRECTA: Repetir la expresión o encapsular en una subconsulta/CTE
SELECT nombre, (precio * 1.19) AS precio_con_iva
FROM productos
WHERE (precio * 1.19) > 100;
```

---

### Operadores de Comparación y Lógicos

| Operador | Significado | Ejemplo |
| :--- | :--- | :--- |
| `=`, `!=` o `<>` | Igual / Diferente | `WHERE estado = 'activo'`, `WHERE rol <> 'invitado'` |
| `<`, `<=`, `>`, `>=` | Comparaciones de orden | `WHERE edad >= 18` |
| `BETWEEN a AND b` | Rango inclusivo ($[a, b]$) | `WHERE precio BETWEEN 10 AND 50` |
| `IN (v1, v2, ...)` | Pertenencia a un conjunto | `WHERE pais IN ('CO', 'MX', 'AR')` |
| `LIKE` | Coincidencia de patrones (`%` comodín, `_` un carácter) | `WHERE email LIKE '%@gmail.com'` |
| `AND`, `OR`, `NOT` | Conectores lógicos booleanos | `WHERE activo = 1 AND (rol = 'admin' OR rol = 'editor')` |

> [!WARNING]
> **Precedencia de Operadores:** `AND` tiene mayor precedencia matemática que `OR`.  
> Siempre utiliza paréntesis al combinar condiciones con `OR` para evitar filtrados erróneos accidentales.

---

### La Trampa de los NULL: Lógica Trivalente (*Three-Valued Logic*)

En SQL, un valor `NULL` no es ni cero, ni una cadena vacía, ni el número `0`: representa la **ausencia de valor o valor desconocido**.

Por esta razón, SQL no opera con lógica binaria tradicional (`TRUE` / `FALSE`), sino con **Lógica Trivalente**:
$$\text{Resultado} \in \{\text{TRUE}, \text{FALSE}, \text{UNKNOWN}\}$$

#### ¿Por qué `columna = NULL` nunca funciona?
Cualquier comparación directa con `NULL` (incluso `NULL = NULL`) se evalúa como **`UNKNOWN`**. Como la cláusula `WHERE` solo deja pasar filas donde la condición sea estrictamente `TRUE`, las condiciones con `=` o `!=` sobre `NULL` descartan la fila silenciosamente.

```sql
-- INCORRECTO: Nunca devolverá ninguna fila
SELECT * FROM usuarios WHERE telefono = NULL;
SELECT * FROM usuarios WHERE telefono != NULL;

-- CORRECTO: Usar los operadores de prueba de nulidad
SELECT * FROM usuarios WHERE telefono IS NULL;
SELECT * FROM usuarios WHERE telefono IS NOT NULL;
```

> [!TIP]
> **Operador `NULL-safe equal` (`<=>` en MySQL):**  
> MySQL incluye el operador `<=>` que evalúa `NULL <=> NULL` como `TRUE` y `valor <=> NULL` como `FALSE` de forma segura:
> ```sql
> SELECT * FROM usuarios WHERE telefono <=> NULL; -- Devuelve las filas con NULL
> ```

---

### Bajo el Capó: Sargabilidad y Uso de Índices

El término **SARGable** proviene de las siglas en inglés ***Search Argument Able*** (Capacidad de servir como argumento de búsqueda para un índice).

Una condición en el `WHERE` es sargable si el optimizador de consultas puede aprovechar un índice B+ Tree existente para realizar una búsqueda binaria en el árbol (*Index Range Scan*) en lugar de recorrer toda la tabla fila por fila (*Full Table Scan*).

#### 1. Trampa: Envolver columnas indexadas en funciones
Cuando aplicas una función sobre una columna en el `WHERE`, el motor no puede adivinar el resultado de antemano y se ve forzado a ejecutar la función fila por fila en toda la tabla:

```sql
-- NO SARGABLE (Destruye el índice sobre 'fecha_registro'):
SELECT * FROM transacciones WHERE YEAR(fecha_registro) = 2026;

-- SARGABLE (Aprovecha el índice B+ Tree en microsegundos):
SELECT * FROM transacciones 
WHERE fecha_registro >= '2026-01-01 00:00:00' 
  AND fecha_registro < '2027-01-01 00:00:00';
```

*(Nota: En MySQL 8.0.13+ es posible crear **Índices Funcionales** para casos inevitables: `CREATE INDEX idx_anio ON transacciones ((YEAR(fecha_registro)));`)*.

#### 2. Trampa: Patrón `LIKE` con comodín a la izquierda
Los índices B+ Tree ordenan los textos de izquierda a derecha (como un diccionario).
```sql
-- SARGABLE: Busca directamente por el prefijo 'admin' en el índice
SELECT * FROM usuarios WHERE username LIKE 'admin%';

-- NO SARGABLE: Requiere escanear toda la tabla fila por fila
SELECT * FROM usuarios WHERE username LIKE '%admin';
```

#### 3. Trampa: Coerción implícita de tipos (Type Mismatch)
Si una columna es de tipo `VARCHAR` pero la comparas con un número entero, MySQL convierte internamente la columna a número para cada fila (`CAST(columna AS DOUBLE)`), invalidando el índice:

```sql
-- Si 'telefono' es VARCHAR(20) y haces:
SELECT * FROM clientes WHERE telefono = 4141234567;   -- NO USA ÍNDICE (coerción implícita)

-- Forma correcta respetando el tipo de dato:
SELECT * FROM clientes WHERE telefono = '4141234567'; -- USA ÍNDICE B+ Tree
```

---

### Buenas Prácticas en el WHERE
1. **La regla de oro de la Sargabilidad:** Mantén siempre la columna indexada "limpia" de funciones, operaciones aritméticas o conversiones a la izquierda del operador (`columna = expresion`, no `f(columna) = expresion`).
2. **Defensa en Profundidad en Filtrado:** En endpoints de APIs o backends, valida y sanea siempre los parámetros de entrada antes de armar la consulta SQL, y utiliza obligatoriamente **sentencias preparadas (*Prepared Statements*)** para evitar inyección SQL (SQLi).
3. **Indexar columnas de alta selectividad:** Si una columna se usa recurrentemente en el `WHERE` y discrimina un alto porcentaje de filas (alta cardinalidad como `email`, `documento_identidad`), debe contar con un índice.

---

## 3. Cláusula DISTINCT

La cláusula `DISTINCT` se utiliza dentro de una sentencia `SELECT` para filtrar y eliminar filas duplicadas en el conjunto de resultados, asegurando que cada fila devuelta sea única.

### Sintaxis y Ejemplos

```sql
SELECT DISTINCT columna1, columna2, ...
FROM nombre_tabla;
```

#### 1. Obtener valores únicos de una columna:
```sql
-- Obtener los años únicos de lanzamiento sin repeticiones
SELECT DISTINCT anio_lanzamiento FROM series;
```

#### 2. DISTINCT con múltiples columnas (Combinaciones únicas):
Cuando se especifican múltiples columnas, `DISTINCT` evalúa la **combinación completa** de los valores seleccionados:
```sql
-- Devuelve combinaciones únicas de ciudad y país
SELECT DISTINCT ciudad, pais FROM clientes;
```

### Perspectiva Matemática y de Programación (Set vs Multiset)

> [!TIP]
> **Conexión con Teoría de Conjuntos y Programación:**  
> En la teoría de conjuntos matemática estricta, un conjunto no contiene elementos repetidos. Sin embargo, SQL por defecto opera sobre **multiconjuntos (multisets o bags)**, permitiendo duplicados en la proyección.  
> Al utilizar `DISTINCT`, forzamos a SQL a comportarse como un verdadero conjunto matemático, equivalente a la estructura de datos **`Set`** en lenguajes como JavaScript o Python.

### Impacto en Rendimiento e Índices (Loose Index Scan vs Temporary Table)

Eliminar duplicados no es una operación gratuita para el motor:

1. **Tablas Temporales (`Using temporary`):** Si la columna consultada no posee un índice, MySQL debe leer todos los registros candidatos, cargarlos en una tabla temporal en memoria (o volcar a disco si supera `tmp_table_size`) y deduplicar antes de emitir la primera fila.
2. **Optimización por Índice (*Loose Index Scan*):** Si la columna cuenta con un índice B+ Tree, el motor puede emplear la optimización *Loose Index Scan* (`Using index for group-by` en el `EXPLAIN`), leyendo de forma directa los valores únicos sin crear tablas temporales en RAM. *(Nota: no confundir con Index Skip Scan, que es una optimización de rangos en índices compuestos)*.

### Criterio de Arquitectura (BD vs Backend)
- **Criterio de Volumen y Red:** Si el `DISTINCT` reduce radicalmente el tráfico de red (ej. pasar de 2.000.000 de filas a solo 4 estados únicos), la deduplicación debe ocurrir **siempre en la base de datos**.
- **Criterio de Delegación al Backend:** Si el conjunto de datos ya es reducido (< 500 filas) y el backend ya tiene los registros en memoria, procesar la deduplicación con un `Set` en la aplicación ahorra ciclos de CPU al motor de base de datos.

---

## 4. Cláusula ORDER BY

Por definición matemática en el modelo relacional, las filas de una tabla representan un conjunto **desordenado**. Si una consulta no incluye explícitamente una cláusula `ORDER BY`, el motor no garantiza ningún orden específico en los resultados devueltos.

### Sintaxis y Dirección (ASC vs DESC)

```sql
SELECT columna1, columna2, ...
FROM nombre_tabla
ORDER BY columna1 [ASC | DESC];
```

- **`ASC` (Ascendente):** De menor a mayor, orden alfabético y de fechas más antiguas a más recientes.
- **`DESC` (Descendente):** De mayor a menor, orden alfabético inverso y fechas más recientes primero.

#### El Matiz del Collation en Textos
En MySQL 8.0 el collation predeterminado es `utf8mb4_0900_ai_ci` (**a**ccent **i**nsensitive, **c**ase **i**nsensitive). Por ello, en una ordenación por defecto `A` y `a` tienen el mismo peso y empatan. Para forzar un ordenamiento donde las mayúsculas vayan antes que las minúsculas según la tabla ASCII (`[A-Z]` antes que `[a-z]`), la columna debe tener o forzarse a una colación binaria (`COLLATE utf8mb4_bin`).

### Ordenamiento Múltiple y Manejo de NULLs

#### 1. Criterios de ordenamiento secuencial:
```sql
SELECT anio_lanzamiento, titulo 
FROM series 
ORDER BY anio_lanzamiento DESC, titulo ASC;
```

#### 2. Comportamiento con valores `NULL`:
- En **MySQL**, los valores `NULL` se consideran computacionalmente como el valor más bajo posible:
  - En orden `ASC`: Los `NULL` aparecen **al principio**.
  - En orden `DESC`: Los `NULL` aparecen **al final**.

> [!TIP]
> **Pro-Tip de Control de NULLs:** Si necesitas ordenar de forma ascendente enviando los `NULL` al final:
> ```sql
> SELECT nombre, telefono 
> FROM usuarios 
> ORDER BY telefono IS NULL ASC, telefono ASC;
> ```
> *(En PostgreSQL existe la cláusula nativa `ORDER BY telefono ASC NULLS LAST`)*.

### Bajo el Capo: Cómo Ordena el Motor (Index Scan vs Filesort)

1. **Ordenamiento por Índice:** Si el `ORDER BY` aprovecha un índice B+ Tree (o la `PRIMARY KEY`), el motor lee las hojas del árbol ya ordenadas. Sin embargo, si es un índice secundario y se requieren más columnas, el optimizador debe balancear el costo de leer el índice vs hacer *lookups* a la tabla principal.
2. **`Using filesort`:** Si no hay índice utilizable, MySQL ordena los registros en un buffer en RAM (`sort_buffer_size`). Aclaración técnica: `Using filesort` **no significa que necesariamente se escriba a disco**; solo va a disco si el volumen a ordenar excede el buffer de memoria asignado.

---

## 5. Cláusula LIMIT y Paginación

La cláusula `LIMIT` se utiliza para restringir el número máximo de filas que devuelve una consulta.

### Sintaxis y Uso Básico

```sql
SELECT columna1, columna2, ...
FROM nombre_tabla
LIMIT cantidad;
```

> [!IMPORTANT]
> **LIMIT sin ORDER BY es no determinista:** Para que un `LIMIT` devuelva resultados consistentes en una API o interfaz, **siempre debe acompañarse de un `ORDER BY`**.

---

### Paginación Tradicional con OFFSET

```sql
-- Sintaxis SQL estándar (Recomendada)
SELECT * FROM productos ORDER BY id ASC LIMIT 10 OFFSET 20;

-- Sintaxis compacta nativa de MySQL: LIMIT [offset], [cantidad]
SELECT * FROM productos ORDER BY id ASC LIMIT 20, 10;
```
*(Salta las primeras 20 filas y retorna las siguientes 10).*

---

### El Problema de Rendimiento: Paginación Profunda (*Deep Paging*)

> **¿Cómo procesa MySQL un `OFFSET 1000000 LIMIT 10`?**  
> Para devolver esas 10 filas, MySQL debe leer físicamente **1.000.010 registros**, descartar el primer millón en memoria y quedarse únicamente con las 10 finales, degradando el tiempo de respuesta conforme aumenta la página.

---

### Solución Profesional: Paginación por Cursor (*Keyset Pagination*)

En APIs modernas de alto rendimiento, se pagina condicionando el `WHERE` por el último identificador visto:

```sql
-- Página 1:
SELECT id, nombre, email 
FROM usuarios 
ORDER BY id ASC 
LIMIT 10;
-- (Supongamos que el último registro devuelto fue id = 10)

-- Página 2 (Sin OFFSET, saltando directo por índice B+ Tree):
SELECT id, nombre, email 
FROM usuarios 
WHERE id > 10 
ORDER BY id ASC 
LIMIT 10;
```

#### Ventajas del Keyset Pagination:
1. **Rendimiento constante $O(1)$:** Salta instantáneamente al valor en el árbol B+ sin procesar filas descartadas.
2. **Inmune a saltos o duplicados:** Las inserciones concurrentes no desplazan las páginas de los usuarios activos.

---

## 6. Funciones de Agregación

Las funciones de agregación operan sobre un conjunto o partición de múltiples valores correspondientes a una columna y computan una reducción matemática para retornar un **único valor escalar representativo**.

Son indispensables para reportes analíticos, paneles de control (dashboards), métricas operativas y cálculos de negocio directamente en el motor de base de datos.

### Concepto y Catálogo Esencial

| Función | Operación Matemática / Lógica | Tipo de Dato Compatible |
| :--- | :--- | :--- |
| `COUNT()` | Cuenta el número de filas o valores no nulos evaluados. | Cualquier tipo de dato. |
| `SUM()` | Suma acumulativa de los valores numéricos. | Numérico (`INT`, `DECIMAL`, `FLOAT`). Retorna `NULL` si el conjunto está vacío o solo contiene nulos. |
| `AVG()` | Calcula la media aritmética ($\frac{\sum x}{n}$). | Numérico. Ignora automáticamente valores `NULL`. |
| `MIN()` | Extrae el valor mínimo o límite inferior del conjunto. | Numéricos, fechas/timestamps y cadenas (orden lexicográfico/collation). |
| `MAX()` | Extrae el valor máximo o límite superior del conjunto. | Numéricos, fechas/timestamps y cadenas (orden lexicográfico/collation). |

---

### Ejemplos Prácticos con Dominio Propio (Plataforma de Vehículos)

#### 1. Cálculo de Totales y Promedios Financieros (`SUM`, `AVG`):
```sql
-- Calcular el inventario valorizado total y el precio medio de venta
SELECT 
    SUM(precio_venta) AS valor_total_inventario,
    AVG(precio_venta) AS precio_promedio_vehiculo
FROM vehiculos;
```

#### 2. Detección de Rangos y Extremos (`MIN`, `MAX`):
```sql
-- Determinar el rango de precios y el modelo más antiguo/reciente disponible
SELECT 
    MIN(precio_venta) AS precio_minimo,
    MAX(precio_venta) AS precio_maximo,
    MIN(anio_fabricacion) AS modelo_mas_antiguo,
    MAX(anio_fabricacion) AS modelo_mas_reciente
FROM vehiculos;
```

---

### `COUNT(*)` vs `COUNT(columna)`: El Manejo Crítico de NULLs

Uno de los errores más frecuentes en análisis y desarrollo es asumir que `COUNT(*)` y `COUNT(columna)` son idénticos.

1. **`COUNT(*)`:**
   - Cuenta el **número total de filas** que satisfacen la consulta, sin importar si contienen columnas vacías o `NULL`.
   - Representa la cardinalidad física del conjunto devuelto.
2. **`COUNT(columna)`:**
   - Cuenta exclusivamente las filas donde esa columna específica **no es `NULL`** (`IS NOT NULL`).
   - Si una columna tiene 100 filas pero 20 tienen `NULL`, `COUNT(columna)` devolverá `80`, mientras que `COUNT(*)` devolverá `100`.
3. **`COUNT(DISTINCT columna)`:**
   - Evalúa y cuenta la cantidad de valores únicos (descartando duplicados y omitiendo `NULLs`).

```sql
-- Auditoría de datos completos:
SELECT 
    COUNT(*) AS total_usuarios_registrados,
    COUNT(telefono) AS usuarios_con_telefono_confirmado,
    COUNT(DISTINCT pais) AS total_paises_operando
FROM usuarios;
```

---

### Uso de Alias Semánticos (`AS`)

Cuando se aplica una función de agregación sin un alias, el motor nombra la columna resultante con la expresión cruda (ej. `SUM(precio_venta)` o `COUNT(*)`).

Para consumir estos datos de forma limpia en APIs, frameworks (como Laravel / Eloquent, Prisma o Express) y componentes de frontend:
- Se utiliza la palabra clave `AS` para definir un identificador semántico en `snake_case`.
- Es una excelente práctica de diseño de software para que el contrato JSON de la API sea autodocumentado.

```sql
SELECT 
    COUNT(*) AS total_ventas,
    SUM(monto_total) AS facturacion_bruta,
    ROUND(AVG(monto_total), 2) AS ticket_promedio
FROM ordenes_compra;
```

---

### Filtrado Previo con `WHERE`

Por el orden lógico de ejecución (`FROM` → `WHERE` → funciones de agregación), las condiciones del `WHERE` restringen las filas **antes** de que la función de agregado realice su cómputo:

```sql
-- Métricas restringidas únicamente a un segmento o concesionario específico
SELECT 
    COUNT(*) AS total_sedanes_disponibles,
    AVG(kilometraje) AS kilometraje_promedio,
    MIN(precio_venta) AS precio_ganga
FROM vehiculos
WHERE tipo_carroceria = 'Sedan' 
  AND estado_publicacion = 'Activo';
```

---

### Impacto en Rendimiento e Índices B+ Tree

1. **Optimización Extrema de `MIN()` y `MAX()`:**
   - Si la columna evaluada cuenta con un índice B+ Tree (o es la `PRIMARY KEY`), calcular `MIN(columna)` o `MAX(columna)` tiene costo computacional **$O(1)$ (tiempo constante en microsegundos)**.
   - En lugar de leer toda la tabla, el motor lee únicamente el primer nodo hoja a la izquierda del árbol para `MIN()` o el último nodo hoja a la derecha para `MAX()` (`Select tables optimized away` en el `EXPLAIN`).
2. **`COUNT(*)` en InnoDB vs MyISAM:**
   - En el antiguo motor MyISAM, `COUNT(*)` se leía instantáneamente de un contador en disco (pero no soportaba transacciones ACID).
   - En **InnoDB (con aislamiento MVCC)**, no existe un contador global estático porque cada transacción concurrente puede ver un número distinto de filas vivas según su snapshot temporal. InnoDB debe escanear el índice secundario más pequeño disponible en memoria para contar las filas vivas.

---

## 7. Cláusula GROUP BY (Agrupamiento de Datos)

La cláusula `GROUP BY` colapsa múltiples filas que comparten los mismos valores en columnas específicas en **grupos únicos de resumen**.

Mientras que las funciones de agregación por sí solas reducen **toda la tabla** a un único valor escalar (ej. la suma total de ventas), `GROUP BY` permite particionar el cálculo para obtener ese valor escalar **por cada categoría o segmento**.

### Concepto y Sintaxis Básica

```sql
SELECT columna_agrupada, FUNCION_AGREGADO(columna_calculo) AS alias
FROM nombre_tabla
WHERE condicion_previa
GROUP BY columna_agrupada;
```

#### Ejemplo: Métrica agrupada por categoría
```sql
-- Obtener la cantidad de vehículos y el precio promedio por tipo de carrocería
SELECT 
    tipo_carroceria,
    COUNT(*) AS total_unidades,
    ROUND(AVG(precio_venta), 2) AS precio_promedio
FROM vehiculos
GROUP BY tipo_carroceria;
```

---

### La Regla de Oro y `ONLY_FULL_GROUP_BY` en MySQL 8.0

> [!CAUTION]
> ### Regla de Oro del GROUP BY
> Toda columna que aparezca en la lista del `SELECT` **debe cumplir una de dos condiciones**:
> 1. Estar presente explícitamente en la cláusula `GROUP BY`.
> 2. Estar envuelta dentro de una función de agregación (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `GROUP_CONCAT`).

#### ¿Por qué? El Error Clásico en MySQL 8.0:
En versiones muy antiguas de MySQL (5.6 o anteriores), si agrupabas por `tipo_carroceria` pero seleccionabas `SELECT tipo_carroceria, color`, MySQL devolvía arbitrariamente el `color` de la primera fila física que encontraba en disco para ese grupo, generando resultados no deterministas y bugs silenciosos.

En **MySQL 8.0+**, el modo SQL estricto **`ONLY_FULL_GROUP_BY`** viene **habilitado por defecto**:

```sql
-- ESTO FALLA en MySQL 8.0+:
-- ERROR 1055 (42000): Expression #2 of SELECT list is not in GROUP BY clause
-- and contains nonaggregated column 'vehiculos.color' which is not functionally 
-- dependent on columns in GROUP BY clause
SELECT tipo_carroceria, color, COUNT(*) 
FROM vehiculos 
GROUP BY tipo_carroceria;

-- FORMA CORRECTA 1: Agregar 'color' al GROUP BY si quieres el desglose por ambos
SELECT tipo_carroceria, color, COUNT(*) 
FROM vehiculos 
GROUP BY tipo_carroceria, color;

-- FORMA CORRECTA 2: Si solo quieres una muestra representativa o agregada
SELECT 
    tipo_carroceria, 
    COUNT(*) AS total,
    GROUP_CONCAT(DISTINCT color ORDER BY color SEPARATOR ', ') AS colores_disponibles
FROM vehiculos 
GROUP BY tipo_carroceria;
```

---

### Agrupación por Múltiples Columnas

Se pueden agrupar datos por jerarquías o combinaciones de múltiples atributos separándolos por comas. El motor crea un grupo por cada **tupla única** de valores:

```sql
-- Conteo y volumen de inventario por marca y año de fabricación
SELECT 
    marca,
    anio_fabricacion,
    COUNT(*) AS unidades_en_stock,
    SUM(precio_venta) AS valor_total_marca_anio
FROM vehiculos
WHERE estado_publicacion = 'Activo'
GROUP BY marca, anio_fabricacion
ORDER BY marca ASC, anio_fabricacion DESC;
```

---

### Manejo de NULLs en el Agrupamiento

- Si la columna sobre la que se agrupa contiene valores `NULL`, el motor **no los descarta**.
- MySQL agrupa todos los registros con `NULL` en **un único grupo conjunto de valor nulo**:

```sql
-- Si algunos clientes no tienen 'ciudad' registrada:
SELECT 
    COALESCE(ciudad, 'Sin Ciudad Asignada') AS ciudad_normalizada,
    COUNT(*) AS total_clientes
FROM clientes
GROUP BY ciudad;
```

---

### Bajo el Capó: Tablas Temporales vs Loose Index Scan

El costo de agrupar depende directamente de la presencia de índices:

1. **Sin Índice (`Using temporary; Using filesort`):**  
   MySQL debe leer todas las filas coincidentes, escribir una tabla temporal en memoria (o volcar a disco en disco si excede `tmp_table_size`) y ordenar las claves del grupo antes de calcular las funciones agregadas.
2. **Con Índice B+ Tree (*Loose Index Scan* / *Tight Index Scan*):**  
   Si agrupas por una columna indexada (o el prefijo izquierdo de un índice compuesto), MySQL recorre el árbol de índice que ya está naturalmente agrupado por clave. En el `EXPLAIN` se observa `Using index for group-by`, ejecutando la consulta con un consumo mínimo de CPU y sin tablas temporales en memoria.

---

## 8. Cláusula HAVING (Filtrado de Grupos Agregados)

La cláusula `HAVING` se utiliza para **filtrar los grupos resultantes** generados por un `GROUP BY`, evaluando condiciones sobre funciones de agregación.

### Diferencia Fundamental: WHERE vs HAVING

| Característica | Cláusula `WHERE` | Cláusula `HAVING` |
| :--- | :--- | :--- |
| **Momento de ejecución** | **Antes** de agrupar (`FROM` → `WHERE` → `GROUP BY`). | **Después** de agrupar (`GROUP BY` → `HAVING`). |
| **Sobre qué opera** | Sobre **filas individuales** de la tabla original. | Sobre **grupos agregados** ya calculados. |
| **¿Acepta funciones agregadas?** | **No**. `WHERE COUNT(*) > 5` arroja error de sintaxis inmediato. | **Sí**. Diseñada específicamente para `HAVING COUNT(*) > 5` o `HAVING SUM(total) >= 1000`. |
| **Uso de índices** | Puede usar índices B+ Tree directamente para evitar escanear la tabla (*sargable*). | Filtra en memoria los grupos ya computados; no usa índices directos para descartar filas base. |

> [!IMPORTANT]
> ### ¿Por qué `WHERE SUM(...)` es ilegal en SQL?
> Porque cuando el motor procesa el `WHERE`, todavía no ha leído todas las filas ni ha computado ninguna suma. La suma no existe todavía en ese punto de la tubería de ejecución.

---

### Sintaxis y Ejemplos de Negocio

```sql
SELECT columna_agrupada, FUNCION_AGREGADO(columna) AS metrica
FROM nombre_tabla
WHERE condicion_filas_individuales
GROUP BY columna_agrupada
HAVING condicion_sobre_agregado;
```

#### Ejemplo 1: Detectar marcas con alto volumen de inventario
```sql
-- Queremos únicamente las marcas que tengan más de 15 vehículos en stock
SELECT 
    marca,
    COUNT(*) AS total_vehiculos,
    AVG(precio_venta) AS precio_promedio
FROM vehiculos
WHERE estado_publicacion = 'Activo'
GROUP BY marca
HAVING COUNT(*) > 15
ORDER BY total_vehiculos DESC;
```

> [!NOTE]
> ### ¿Se puede usar el alias de columna en el HAVING? (`HAVING total_vehiculos > 15`)
> En **MySQL**, esta sintaxis alternativa es **100% válida y permitida**:
> ```sql
> SELECT 
>     marca,
>     COUNT(*) AS total_vehiculos
> FROM vehiculos
> WHERE estado_publicacion = 'Activo'
> GROUP BY marca
> HAVING total_vehiculos > 15  -- ¡Válido en MySQL!
> ORDER BY total_vehiculos DESC;
> ```
> **⚠️ Advertencia Crítica de Portabilidad (MySQL vs PostgreSQL / Estándar ANSI SQL):**  
> - Según el orden lógico formal del estándar SQL ANSI, `HAVING` se procesa **antes** que `SELECT`, por lo que los alias teóricamente aún no existen en esa fase.
> - **MySQL** ofrece una extensión sintáctica permisiva y resuelve el alias del `SELECT` sin problemas.
> - Sin embargo, en **PostgreSQL**, **Oracle** o **SQL Server**, escribir `HAVING total_vehiculos > 15` fallará inmediatamente con:
>   ```text
>   ERROR: column "total_vehiculos" does not exist
>   LINE 7: HAVING total_vehiculos > 15
>   ```
> - *(Nota de arquitectura cruzada: Cuando documentemos el módulo de PostgreSQL, realizaremos el benchmark de esta misma consulta para evidenciar el fallo y la necesidad de escribir la función explícita `HAVING COUNT(*) > 15` o encapsular en CTEs/subconsultas).*

#### Ejemplo 2: Auditoría y Detección de Duplicados
Un caso de uso indispensable para DBAs e ingenieros de datos es identificar registros duplicados en tablas que carecen de restricciones únicas:

```sql
-- Encontrar correos electrónicos duplicados en la tabla de usuarios
SELECT 
    email,
    COUNT(*) AS ocurrencias
FROM usuarios
GROUP BY email
HAVING COUNT(*) > 1;
```

#### Ejemplo 3: Filtrar por agregación monetaria
```sql
-- Concesionarios cuyas ventas acumuladas superan los $500,000 en el año actual
SELECT 
    concesionario_id,
    SUM(monto_total) AS facturacion_total
FROM ventas
WHERE fecha_venta >= '2026-01-01'
GROUP BY concesionario_id
HAVING SUM(monto_total) >= 500000;
```

---

### Buenas Prácticas y Rendimiento: Filtrar Temprano (*Pushdown*)

> [!TIP]
> **Regla de Optimización de Oro:**  
> **Nunca pongas en el `HAVING` una condición que pueda resolverse en el `WHERE`.**

Técnicamente, SQL permite poner condiciones sobre columnas normales en el `HAVING` (por ejemplo: `HAVING estado = 'Activo'`). Sin embargo, hacer esto es un **grave antipatrón de rendimiento**:

- **Mal (Filtrado tardío en `HAVING`):**
  ```sql
  -- ANTIPATRÓN: El motor agrupa TODOS los vehículos (incluso inactivos y vendidos) 
  -- consumiendo RAM y CPU, y descarta los grupos al final.
  SELECT marca, COUNT(*) 
  FROM vehiculos
  GROUP BY marca, estado_publicacion
  HAVING estado_publicacion = 'Activo';
  ```
- **Bien (Filtrado temprano en `WHERE`):**
  ```sql
  -- ÓPTIMO: El motor descarta las filas no deseadas antes de agrupar usando índices.
  -- El GROUP BY procesa únicamente una fracción pequeña de datos.
  SELECT marca, COUNT(*) 
  FROM vehiculos
  WHERE estado_publicacion = 'Activo'
  GROUP BY marca;
  ```


