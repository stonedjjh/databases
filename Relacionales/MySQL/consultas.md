# Consultas en MySQL (Queries)

![Engine](https://img.shields.io/badge/Engine-MySQL-4479A1?style=flat&logo=mysql&logoColor=white)
![Focus](https://img.shields.io/badge/Focus-DML%20%26%20Retrieval-blue)

Guia practica sobre el diseno, ejecucion y optimizacion de consultas para la extraccion de datos.

---

## Que es una Consulta (Query)

Una consulta (query) es una instruccion o solicitud formal que se envia al motor de base de datos para recuperar, filtrar, ordenar o transformar informacion almacenada en una o varias tablas. 

En bases de datos relacionales, las consultas de lectura se construyen mediante el lenguaje declarativo SQL: el usuario especifica **que** datos necesita y bajo que condiciones, mientras que el optimizador del motor decide **como** acceder fisicamente a los registros (a traves de indices o lecturas secuenciales de disco).

> [!NOTE]
> Las bases de datos relacionales estan fuertemente ligadas a la teoria de conjuntos. Aunque no es indispensable dominar matematicas avanzadas para aprender y escribir SQL, a lo largo de esta guia iremos explicando de forma sencilla como se vinculan estos conceptos para entender mejor que hace el motor tras bambalinas.

---

## Indice de Contenido

- [1. Clausula SELECT](#1-clausula-select)
  - [Sintaxis Basica](#sintaxis-basica)
  - [Ejemplos de Uso](#ejemplos-de-uso)
  - [Buenas Practicas](#buenas-practicas)
- [2. Clausula DISTINCT](#2-clausula-distinct)
  - [Sintaxis y Ejemplos](#sintaxis-y-ejemplos)
  - [Perspectiva Matematica y de Programacion (Set vs Multiset)](#perspectiva-matematica-y-de-programacion-set-vs-multiset)
  - [Impacto en Rendimiento e Indices](#impacto-en-rendimiento-e-indices)
- [3. Clausula ORDER BY](#3-clausula-order-by)
  - [Sintaxis y Direccion (ASC vs DESC)](#sintaxis-y-direccion-asc-vs-desc)
  - [Ordenamiento Multiple y Manejo de NULLs](#ordenamiento-multiple-y-manejo-de-nulls)
  - [Bajo el Capo: Como Ordena el Motor (Index Scan vs Filesort)](#bajo-el-capo-como-ordena-el-motor-index-scan-vs-filesort)
- [4. Clausula LIMIT y Paginacion](#4-clausula-limit-y-paginacion)
  - [Sintaxis y Uso Basico](#sintaxis-y-uso-basico)
  - [Paginacion Tradicional con OFFSET](#paginacion-tradicional-con-offset)
  - [El Problema de Rendimiento: Paginacion Profunda (Deep Paging)](#el-problema-de-rendimiento-paginacion-profunda-deep-paging)
  - [Solucion Profesional: Paginacion por Cursor (Keyset Pagination)](#solucion-profesional-paginacion-por-cursor-keyset-pagination)

---

## 1. Clausula SELECT

La sentencia `SELECT` es el comando fundamental de SQL utilizado para leer y recuperar registros desde la base de datos.

### Sintaxis Basica

```sql
SELECT columna1, columna2, ...
FROM nombre_tabla;
```

### Ejemplos de Uso

1. **Consultar todas las columnas de una tabla:**

   ```sql
   SELECT * FROM usuarios;
   ```

   > [!IMPORTANT]
   > El caracter asterisco (`*`) actua como un comodin (*wildcard*) que selecciona todos los campos y columnas disponibles en las tablas involucradas dentro de la consulta.

2. **Consultar columnas especificas (recomendado):**

   ```sql
   SELECT id, nombre, email FROM usuarios;
   ```

3. **Uso de alias para renombrar columnas en la salida:**

   ```sql
   SELECT nombre AS nombre_completo, email AS correo_contacto 
   FROM usuarios;
   ```

4. **Consultas sin tabla (expresiones o valores directos):**

   ```sql
   SELECT 10 + 5 AS resultado;
   ```

> [!TIP]
> **Dato de valor:** Desde la perspectiva de la teoria de conjuntos, al ejecutar un `SELECT` estamos seleccionando un subconjunto de atributos (columnas) de la tabla. En algebra relacional, esta operacion se conoce formalmente como **proyeccion**.

### Buenas Practicas

- **Mayusculas en palabras clave (Case-Insensitive):** Los comandos y clausulas SQL no distinguen entre mayusculas y minusculas (`SELECT` es funcionalmente equivalente a `select`). Sin embargo, por estandar y legibilidad se escriben siempre en **MAYUSCULAS** para diferenciarlas inmediatamente de los identificadores de tablas y columnas.
- **Sensibilidad de nombres segun el Sistema Operativo (Windows vs Linux):** Aunque las palabras clave son insensibles, en MySQL los nombres de tablas y bases de datos corresponden a directorios y archivos fisicos en disco (`.ibd`):
  - En **Windows**, el sistema de archivos no distingue mayusculas/minusculas por defecto (`usuarios` y `Usuarios` apuntan al mismo archivo).
  - En **Linux** (donde corre la mayoria de servidores y contenedores Docker en produccion), el sistema de archivos **si** distingue entre mayusculas y minusculas.
  - *Impacto:* Una consulta o migracion con `FROM Usuarios` funcionara en desarrollo en Windows pero fallara en Linux con un error de tabla no encontrada. Por ello, la regla de oro es nombrar siempre tablas y columnas en **minusculas utilizando snake_case** (`usuarios_registrados`).
- **Evitar `SELECT *` en entornos de produccion y APIs:** Solicitar solo los campos requeridos reduce el trafico de red, optimiza el consumo de memoria del servidor de base de datos y permite que el motor aproveche indices de cobertura (*covering indexes*).
- **Proyeccion consciente:** Seleccionar explicitamente las columnas evita que cambios en la estructura de la tabla (como agregar o mover columnas) rompan el mapeo de datos en el backend.

---

## 2. Clausula DISTINCT

La clausula `DISTINCT` se utiliza dentro de una sentencia `SELECT` para filtrar y eliminar filas duplicadas en el conjunto de resultados, asegurando que cada fila devuelta sea unica.

### Sintaxis y Ejemplos

```sql
SELECT DISTINCT columna1, columna2, ...
FROM nombre_tabla;
```

#### 1. Obtener valores unicos de una columna:

```sql
-- Obtener los años unicos de lanzamiento sin repeticiones
SELECT DISTINCT año_lanzamiento FROM series;
```

#### 2. DISTINCT con multiples columnas (Combinaciones unicas):

Cuando se especifican multiples columnas, `DISTINCT` evalua la **combinacion completa** de los valores seleccionados, no cada columna por separado:

```sql
-- Devuelve combinaciones unicas de ciudad y pais
SELECT DISTINCT ciudad, pais FROM clientes;
```

### Perspectiva Matematica y de Programacion (Set vs Multiset)

> [!TIP]
> **Conexion con Teoria de Conjuntos y Programacion:**
> En la teoria de conjuntos matematica estricta, un conjunto no contiene elementos repetidos. Sin embargo, SQL por defecto opera sobre **multiconjuntos (multisets o bags)**, permitiendo duplicados en la proyeccion.
> 
> Al utilizar `DISTINCT`, forzamos a SQL a comportarse como un verdadero conjunto matematico, exactamente equivalente a la estructura de datos **`Set`** en lenguajes de programacion modernos (como `Set` en JavaScript o Python), donde cualquier elemento repetido es descartado automaticamente.

### Impacto en Rendimiento e Indices

Eliminar duplicados no es una operacion gratuita para el motor de base de datos:

1. **Tablas Temporales y Ordenamiento en Memoria (*Filesort*):** Si la columna consultada no posee un indice, MySQL debe leer todos los registros candidatos, cargarlos en una tabla temporal en memoria (o volcar a disco si supera el buffer permitido) y aplicar un algoritmo de ordenamiento/hashing para descartar duplicados antes de emitir la primera fila.
2. **Optimizacion con Indices (B+ Tree):** Si la columna cuenta con un indice, el arbol B+ ya almacena los valores ordenados contiguamente. En este escenario, el optimizador de MySQL puede aplicar tecnicas como *Index Skip Scan*, leyendo directamente las claves unicas del indice sin necesidad de procesar tablas temporales en RAM.
3. **Criterio de Arquitectura y Trade-offs Modernos (BD vs Backend):**
   - **Regla del Volumen y Red:** Si el `DISTINCT` reduce radicalmente el trafico de red (por ejemplo, pasar de 2.000.000 de filas a solo 4 estados unicos), la deduplicacion debe ocurrir **siempre en la base de datos**. Traer millones de filas al backend para filtrarlas con un `Set` satura el ancho de banda y la memoria de la aplicacion.
   - **Regla de Delegacion al Backend:** Si el conjunto de datos ya es reducido (< 500 filas) o la aplicacion ya cargo esos registros en memoria para otros propositos de la vista, es preferible deduplicar en el backend con estructuras como `Set` o `.filter()`. Esto evita sobrecargar la CPU del servidor de base de datos y aprovecha la escalabilidad horizontal de las instancias de backend.
   - **El contexto de Microservicios (*Database per Service*):** A diferencia de los esquemas monoliticos tradicionales donde la base de datos centralizaba toda la logica y transformaciones, en arquitecturas distribuidas los esquemas son compactos y especializados, trasladando agregaciones y transformaciones ligeras hacia la capa de aplicacion u orquestacion.

---

## 3. Clausula ORDER BY

Por definicion matematica en el modelo relacional, las filas de una tabla representan un conjunto **desordenado**. Si una consulta no incluye explicitamente una clausula `ORDER BY`, el motor de base de datos no garantiza ningun orden especifico en los resultados devueltos (incluso si en lecturas consecutivas parecen venir en el mismo orden).

La clausula `ORDER BY` se utiliza para ordenar el conjunto de resultados segun una o mas columnas.

### Sintaxis y Direccion (ASC vs DESC)

```sql
SELECT columna1, columna2, ...
FROM nombre_tabla
ORDER BY columna1 [ASC | DESC];
```

- **`ASC` (Ascendente):** Orden predeterminado.
  - **Numeros:** De menor a mayor (1, 2, 10, 100).
  - **Cadenas de texto:** Orden alfabetico segun el mapa de caracteres (*Collation*). En ordenamiento binario/ASCII estricto, las mayusculas van antes que las minusculas `[A-Z, a-z]` porque `A` es 65 y `a` es 97.
  - **Fechas:** De mas antiguo a mas reciente.
- **`DESC` (Descendente):** De mayor a menor, orden inverso y de mas reciente a mas antiguo.

#### Ejemplos:

```sql
-- Orden ascendente por defecto (1, 2, 3...)
SELECT episodio_id, titulo FROM episodios ORDER BY episodio_id;

-- Orden descendente explicito (los mas recientes primero)
SELECT titulo, fecha_estreno FROM episodios ORDER BY fecha_estreno DESC;
```

> [!TIP]
> **Datazo 1: El orden lexicografico (Cuando los numeros son texto):**
> Si un campo numerico fue guardado por error como tipo texto (`VARCHAR`), el orden no sera matematico (1, 2, 10), sino lexicografico caracter por caracter:
> ```text
> '1', '10', '100', '2', '25', '3'
> ```
> *Impacto:* '10' aparece antes que '2' porque el caracter '1' es menor que el '2'. Para corregir esto temporalmente en una consulta se puede forzar un casteo: `ORDER BY CAST(columna AS UNSIGNED) ASC`.

> [!NOTE]
> **Datazo 2: ¿Por que antes se evitaba la 'ñ' (`año` vs `anio`) y como se resolvio?**
> En decadas anteriores, las bases de datos utilizaban codificaciones de 7 u 8 bits como **ASCII** o **latin1 / ISO-8859-1**. Un caracter no estandar en ingles como la `ñ`, los acentos o caracteres especiales provocaban corrupcion de caracteres (*mojibake*: `aÃ±o`), errores de migracion y fallos en drivers de conexion de distintos lenguajes. Por ello, la vieja escuela desarrollo la costumbre defensiva de escribir `anio`.
> 
> En la actualidad, con el estandar universal **UTF-8 completo (`utf8mb4`)** adoptado por defecto en MySQL 8.0, nombrar columnas o almacenar valores con `ñ`, tildes o incluso emojis es 100% nativo, seguro y sin problemas de corrupcion.

### Ordenamiento Multiple y Manejo de NULLs

#### 1. Criterios de ordenamiento secuencial:

Se pueden especificar multiples columnas separadas por comas. El motor ordena por la primera columna y, solo en caso de empate (*tie-breaker*), aplica el orden de la siguiente:

```sql
-- Ordena primero por año descendente; si empatan, por titulo alfabetico
SELECT año_lanzamiento, titulo 
FROM series 
ORDER BY año_lanzamiento DESC, titulo ASC;
```

#### 2. Comportamiento con valores `NULL`:

- En **MySQL**, los valores `NULL` se consideran computacionalmente como el valor mas bajo posible.
- Por lo tanto:
  - En orden `ASC`: Los `NULL` aparecen **al principio**.
  - En orden `DESC`: Los `NULL` aparecen **al final**.

> [!TIP]
> **Pro-Tip de Control de NULLs:** Si necesitas ordenar de forma ascendente pero enviando los `NULL` al final sin alterar la columna, puedes utilizar una expresion condicional:
> ```sql
> -- Los NULL iran al final aun en orden ascendente
> SELECT nombre, telefono 
> FROM usuarios 
> ORDER BY telefono IS NULL ASC, telefono ASC;
> ```

### Bajo el Capo: Como Ordena el Motor (Index Scan vs Filesort)

Ordenar registros consume recursos criticos de CPU y memoria:

1. **Ordenamiento por Indice (Zero Cost Sort):** Si la columna en el `ORDER BY` cuenta con un indice B-Tree (o coincide con la clave primaria), el motor no realiza ningun calculo de ordenamiento: lee las hojas del arbol que ya estan fisicamente ordenadas. Es la forma mas rapida posible.
2. **El temido `Using filesort`:** Si la columna no esta indexada, MySQL debe leer las filas, cargarlas en un buffer de memoria RAM asignado por sesion (`sort_buffer_size`) y ejecutar un algoritmo de ordenamiento (generalmente *quicksort* o *mergesort*). Si los datos superan el tamano de ese buffer, MySQL se ve obligado a crear archivos temporales en disco, degradando severamente el tiempo de respuesta.
3. **Buenas Practicas:** En tablas de produccion con alta concurrencia, toda consulta frecuente que utilice `ORDER BY` sobre un volumen significativo de datos debe estar respaldada por un indice que cubra el ordenamiento.

---

## 4. Clausula LIMIT y Paginacion

La clausula `LIMIT` se utiliza para restringir el numero maximo de filas que devuelve una consulta. Es una herramienta indispensable en el desarrollo web y APIs para implementar paginacion de resultados, proteger el servidor de sobrecargas de memoria y optimizar el tiempo de respuesta.

### Sintaxis y Uso Basico

```sql
SELECT columna1, columna2, ...
FROM nombre_tabla
LIMIT cantidad;
```

#### Ejemplo: Obtener el "Top N"

```sql
-- Obtener las 5 series mas recientes
SELECT titulo, fecha_estreno 
FROM series 
ORDER BY fecha_estreno DESC 
LIMIT 5;
```

> [!IMPORTANT]
> **LIMIT sin ORDER BY es no determinista:** Si ejecutas un `LIMIT 5` sin `ORDER BY`, el motor te devolvera 5 filas arbitrarias segun como las encuentre en las paginas de disco. Para que un `LIMIT` tenga sentido de negocio consistente, **siempre debe acompanarse de un `ORDER BY`**.

---

### Paginacion Tradicional con OFFSET

Para saltar filas y navegar entre paginas de resultados, `LIMIT` acepta dos parametros: o bien la sintaxis con la palabra clave `OFFSET`, o bien la sintaxis compacta de MySQL separada por coma:

```sql
-- Sintaxis SQL estandar (Recomendada por claridad)
SELECT * FROM productos ORDER BY id ASC LIMIT 10 OFFSET 20;

-- Sintaxis abreviada nativa de MySQL: LIMIT [offset], [cantidad]
SELECT * FROM productos ORDER BY id ASC LIMIT 20, 10;
```
*(Ambas sentencias significan: "Salta los primeros 20 registros y devuelve los siguientes 10").*

#### Formula para APIs y Backend:
- `OFFSET = (pagina - 1) * elementos_por_pagina`
- Pagina 1: `LIMIT 10 OFFSET 0`
- Pagina 2: `LIMIT 10 OFFSET 10`
- Pagina 3: `LIMIT 10 OFFSET 20`

---

### El Problema de Rendimiento: Paginacion Profunda (*Deep Paging*)

Aunque `LIMIT ... OFFSET` es el metodo mas ensenado en tutoriales basicos, tiene una **falla critica de arquitectura y escalabilidad**:

> **¿Como procesa MySQL un `OFFSET 1000000 LIMIT 10`?**
> Para devolverte esas 10 filas de la pagina lejana, MySQL **no salta por arte de magia**. El motor tiene que:
> 1. Leer físicamente **1.000.010 registros** del disco.
> 2. Mantenerlos temporalmente en memoria o descartarlos uno por uno.
> 3. Quedarse unicamente con los ultimos 10.

En tablas grandes con cientos de miles o millones de filas, un usuario o bot que intente ver la pagina 500 hara que la consulta tarde 4 o 8 segundos, devorando la CPU y disco del servidor.

---

### Solucion Profesional: Paginacion por Cursor (*Keyset Pagination*)

En APIs modernas de alto rendimiento (como las de Twitter, Facebook, Stripe o Slack), en lugar de usar `OFFSET`, se utiliza la tecnica de **Keyset Pagination** (o paginacion basada en cursor).

En lugar de decir *"salta 20 filas"*, le decimos al motor *"dame las 10 filas cuyo ID sea mayor al ultimo ID que vi en la pagina anterior"*:

```sql
-- Pagina 1:
SELECT id, nombre, email 
FROM usuarios 
ORDER BY id ASC 
LIMIT 10;
-- (Supongamos que el ultimo registro devuelto de esta pagina fue el id = 10)

-- Pagina 2 (Sin OFFSET, aprovechando el indice de la clave primaria):
SELECT id, nombre, email 
FROM usuarios 
WHERE id > 10 
ORDER BY id ASC 
LIMIT 10;
```

#### Ventajas del Keyset Pagination:
1. **Rendimiento constante O(1):** No importa si estas en la pagina 1 o en la pagina 100.000, MySQL utiliza directamente el arbol B+ del indice primario y salta instantaneamente a la fila `id > X` en microsegundos.
2. **Inmune a saltos o duplicados por inserciones concurrentes:** Si un usuario nuevo se registra mientras otro pagina con `OFFSET`, los registros se desplazan y el usuario ve datos repetidos. Con cursores (`id > 10`), la lista se mantiene estable.
