# Día 5: Consultas SQL (Nivel Intermedio)

![Engine](https://img.shields.io/badge/Engine-MySQL%208.0+-4479A1?style=flat&logo=mysql&logoColor=white)
![Focus](https://img.shields.io/badge/Focus-DML%20Intermedio%20%26%20Funciones-blue)
![Database](https://img.shields.io/badge/Database-Netflix-red?logo=netflix)

Guía rápida y ejercicios prácticos del **Día 5** correspondientes al curso *SQL TOTAL: Domina Bases de Datos de 0 a Avanzado en 12 Días*.

---

## 📑 Temario y Conceptos Clave

1. [Subconsultas (Subqueries)](#1-subconsultas-subqueries)
2. [Función Condicional IF()](#2-función-condicional-if)
3. [Función Condicional CASE](#3-función-condicional-case)
4. [Función de Conversión CAST()](#4-función-de-conversión-cast)
5. [Funciones de FECHA](#5-funciones-de-fecha)
6. [Manipulación de Cadenas de Texto](#6-manipulación-de-cadenas-de-texto)
7. [Funciones Matemáticas](#7-funciones-matemáticas)
8. [Laboratorio Evaluado y Proyecto del Día 5](#8-laboratorio-evaluado-y-proyecto-del-día-5)

> [!NOTE]
> Para explicaciones teóricas exhaustivas, fundamentos de álgebra relacional, sargabilidad, optimizaciones internas del motor InnoDB y planes de ejecución, consulta la guía maestra: [consultas.md](file:///c:/Bases%20de%20Datos/Relacionales/MySQL/consultas.md).

---

## 1. Subconsultas (Subqueries)

Una subconsulta es una consulta anidada dentro de otra. Puede devolver un valor escalar, una fila o un conjunto de filas.

### Ejemplo con Base de Datos Netflix (Series y Episodios)
Calcular el rating promedio por serie y filtrar aquellas series cuyo promedio supera el promedio general de toda la plataforma:

```sql
SELECT 
    s.titulo,
    s.anio_lanzamiento,
    s.genero,
    ROUND(AVG(e.rating_imdb), 2) AS rating_promedio
FROM series s
INNER JOIN episodios e ON s.serie_id = e.serie_id
GROUP BY s.serie_id, s.titulo, s.anio_lanzamiento, s.genero
HAVING AVG(e.rating_imdb) > (
    -- Subconsulta escalar: Promedio global de rating de todos los episodios
    SELECT AVG(rating_imdb) 
    FROM episodios
)
ORDER BY rating_promedio DESC;
```

---

## 2. Función Condicional IF()

Permite evaluar una condición simple de tipo `IF(condicion, si_verdadero, si_falso)`.

```sql
SELECT 
    titulo,
    rating_imdb,
    IF(rating_imdb >= 8.5, 'Excelente / Obra Maestra', 'Estándar') AS clasificacion
FROM episodios;
```

---

## 3. Función Condicional CASE

El estándar ANSI para múltiples condiciones y clasificaciones en rangos.

```sql
SELECT 
    titulo,
    rating_imdb,
    CASE 
        WHEN rating_imdb >= 9.0 THEN 'Imprescindible'
        WHEN rating_imdb >= 8.0 THEN 'Muy Buena'
        WHEN rating_imdb >= 7.0 THEN 'Aceptable'
        ELSE 'Baja Calificación'
    END AS categoria_calidad
FROM episodios;
```

---

## 4. Función de Conversión CAST()

Convierte explícitamente un tipo de dato a otro compatible (`CHAR`, `DATE`, `DATETIME`, `DECIMAL`, `SIGNED`, `UNSIGNED`, `TIME`).

```sql
SELECT 
    titulo,
    CAST(rating_imdb AS DECIMAL(3, 1)) AS rating_un_decimal,
    CAST(anio_lanzamiento AS CHAR(4)) AS anio_texto
FROM series;
```

---

## 5. Funciones de FECHA

Manejo de fechas, componentes y cálculos de intervalos temporales:

- `CURDATE()` / `NOW()`: Fecha y fecha/hora actual.
- `YEAR()`, `MONTH()`, `DAY()`: Extracción de componentes numéricos.
- `DATE_ADD(fecha, INTERVAL n unidad)` / `DATE_SUB(...)`: Suma/resta de periodos.
- `DATEDIFF(f1, f2)`: Diferencia en días.
- `DATE_FORMAT(fecha, '%d/%m/%Y')`: Formateo de fecha.

```sql
SELECT 
    titulo,
    fecha_estreno,
    YEAR(fecha_estreno) AS anio_estreno,
    TIMESTAMPDIFF(YEAR, fecha_estreno, CURDATE()) AS anios_desde_estreno
FROM episodios;
```

---

## 6. Manipulación de Cadenas de Texto

Funciones para transformar y limpiar strings:

- `UPPER(str)` / `LOWER(str)`: Mayúsculas / Minúsculas.
- `SUBSTRING(str, pos, len)` / `LEFT(str, n)` / `RIGHT(str, n)`: Extracción de caracteres (índice base 1).
- `CONCAT(s1, s2, ...)` / `CONCAT_WS(sep, s1, s2, ...)`: Concatenación segura.
- `TRIM(str)` / `REPLACE(str, buscar, reemplazar)`: Sanitización.
- `CHAR_LENGTH(str)`: Cantidad de caracteres (recomendado para UTF-8).

```sql
SELECT 
    UPPER(titulo) AS titulo_mayus,
    SUBSTRING(titulo, 1, 10) AS titulo_corto,
    CONCAT_WS(' - ', s.titulo, e.titulo) AS serie_y_episodio
FROM series s
INNER JOIN episodios e ON s.serie_id = e.serie_id;
```

---

## 7. Funciones Matemáticas

Cálculos numéricos y redondeos:

- `ROUND(x, d)`: Redondeo matemático tradicional a `d` decimales.
- `CEILING(x)` / `CEIL(x)`: Redondeo hacia arriba al entero más próximo.
- `FLOOR(x)`: Redondeo hacia abajo al entero más próximo.
- `ABS(x)`: Valor absoluto.
- `MOD(n, m)`: Resto o residuo de la división.
- `RAND()`: Número flotante pseudoaleatorio entre 0 y 1.

```sql
SELECT 
    titulo,
    rating_imdb,
    ROUND(rating_imdb, 1) AS rating_redondeado,
    CEILING(rating_imdb) AS techo_entero,
    FLOOR(rating_imdb) AS piso_entero
FROM episodios;
```

---

## 8. Laboratorio Evaluado y Proyecto del Día 5

### Reportes de ventas de una cafetería / Consultas agregadas con filtro
- Cálculo de métricas promedio con `ROUND()`.
- Clasificación de tickets o ratings con `CASE WHEN`.
- Extracción de fechas con `YEAR()` y `MONTH()` respetando la sargabilidad en los filtros `WHERE`.
- Agrupación por identificadores primarios (`serie_id` o `producto_id`) y ordenamiento descendente del promedio calculado.
