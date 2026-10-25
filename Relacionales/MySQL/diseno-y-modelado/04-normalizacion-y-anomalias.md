# 04. Normalización, Formas Normales y Anomalías

![Engine](https://img.shields.io/badge/Engine-MySQL%208.0+-4479A1?style=flat&logo=mysql&logoColor=white)
![Focus](https://img.shields.io/badge/Focus-Database%20Normalization%201NF%20to%203NF%20%26%20BCNF-blue)

La **Normalización** es un proceso sistemático y formal de diseño relacional propuesto originalmente por Edgar F. Codd. Su propósito es **eliminar la redundancia de datos**, **garantizar la consistencia lógica** y **erradicar las anomalías operativas** de inserción, actualización y borrado.

---

## Índice

- [1. Las Tres Pesadillas: Anomalías en Bases de Datos Desnormalizadas](#1-las-tres-pesadillas-anomalías-en-bases-de-datos-desnormalizadas)
  - [Anomalía de Inserción](#anomalía-de-inserción)
  - [Anomalía de Actualización (Inconsistencia)](#anomalía-de-actualización-inconsistencia)
  - [Anomalía de Borrado (Pérdida Accidental de Información)](#anomalía-de-borrado-pérdida-accidental-de-información)
- [2. Primera Forma Normal (1NF) - Atomicidad](#2-primera-forma-normal-1nf---atomicidad)
  - [Reglas Matemáticas de 1NF](#reglas-matemáticas-de-1nf)
  - [El Antipatrón del String con Comas y Arrays](#el-antipatrón-del-string-con-comas-y-arrays)
- [3. Segunda Forma Normal (2NF) - Dependencia Funcional Completa](#3-segunda-forma-normal-2nf---dependencia-funcional-completa)
  - [Dependencia Parcial sobre Claves Compuestas](#dependencia-parcial-sobre-claves-compuestas)
  - [Cómo Descomponer para Alcanzar 2NF](#cómo-descomponer-para-alcanzar-2nf)
- [4. Tercera Forma Normal (3NF) - Dependencia Transitiva](#4-tercera-forma-normal-3nf---dependencia-transitiva)
  - [Dependencia No Clave -> No Clave](#dependencia-no-clave---no-clave)
  - [La Regla Mnemotécnica de Bill Kent](#la-regla-mnemotécnica-de-bill-kent)
- [5. Forma Normal de Boyce-Codd (BCNF)](#5-forma-normal-de-boyce-codd-bcnf)
- [6. ¿Cuándo y Por Qué Desnormalizar en Producción?](#6-cuándo-y-por-qué-desnormalizar-en-producción)
  - [OLTP (Transaccional) vs OLAP (Analítico / Data Warehouse)](#oltp-transaccional-vs-olap-analítico--data-warehouse)
  - [Snapshots Históricos Obligatorios (Inmutabilidad Financiera)](#snapshots-históricos-obligatorios-inmutabilidad-financiera)

---

## 1. Las Tres Pesadillas: Anomalías en Bases de Datos Desnormalizadas

Consideremos la siguiente tabla no normalizada ("hoja de Excel típica") donde una empresa guarda pedidos:

| id_pedido | cliente_nombre | cliente_ciudad | cliente_telefono | producto_nombre | producto_categoria | precio |
| :---: | :--- | :--- | :--- | :--- | :--- | :---: |
| 1 | Carlos Gómez | Bogotá | 3001234567 | Laptop Pro | Electrónica | 1200 |
| 2 | Ana Martínez | Medellín | 3109876543 | Mouse Óptico | Accesorios | 25 |
| 3 | Carlos Gómez | Bogotá | 3001234567 | Teclado Mecánico| Accesorios | 80 |

### Anomalía de Inserción
- **Problema:** Si la empresa adquiere un nuevo producto (ej. "Monitor 4K") o registra un nuevo cliente que aún no ha comprado nada, **no puede insertarlo en la base de datos** porque el `id_pedido` sería `NULL` (violando la clave primaria).

### Anomalía de Actualización
- **Problema:** Si Carlos Gómez cambia de número telefónico, debemos actualizar múltiples registros. Si por un fallo de red o error de aplicación se actualiza la fila 1 pero no la fila 3, **la base de datos queda corrupta e inconsistente** (dos teléfonos distintos para la misma persona).

### Anomalía de Borrado
- **Problema:** Si Ana Martínez decide cancelar el pedido 2 y borramos esa fila, **perdemos completamente de la existencia a Ana Martínez y sus datos de contacto** del sistema.

---

## 2. Primera Forma Normal (1NF) - Atomicidad

Una tabla está en **1NF** si y solo si:
1. Cada celda contiene un **único valor atómico e indivisible** (sin listas, sin arrays, sin campos separados por comas).
2. No existen **grupos repetitivos** de columnas (ej. `telefono_1`, `telefono_2`, `telefono_3`).
3. Existe una **Clave Primaria** definida que identifica inequívocamente cada fila.

### El Antipatrón del String con Comas

```sql
-- ANTIPATRÓN CRÍTICO (Viola 1NF):
CREATE TABLE empleados (
    id INT PRIMARY KEY,
    nombre VARCHAR(100),
    telefonos VARCHAR(255) -- '300123, 310456, 315789' <-- PROHIBIDO
);
```
> [!CAUTION]
> Guardar valores separados por comas impide indexar búsquedas, imposibilita validar integridad con claves foráneas y obliga a usar funciones costosas de parsing de texto como `LIKE '%300123%'`.
> 
> **Solución 1NF:** Crear una tabla independiente `empleados_telefonos` con relación `1:N`.

---

## 3. Segunda Forma Normal (2NF) - Dependencia Funcional Completa

Una tabla está en **2NF** si:
1. Ya cumple con los requisitos de la **1NF**.
2. **Todos los atributos no clave dependen funcionalmente de la totalidad de la Clave Primaria**, no solo de una parte de ella (aplica para tablas con claves primarias compuestas).

### Ejemplo de Violación de 2NF

Consideremos la tabla de una tienda con clave compuesta `(pedido_id, producto_id)`:

```
Clave Primaria: [pedido_id, producto_id]
Atributos:
- cantidad                    --> Depende de AMBOS (correcto)
- producto_nombre             --> Depende SÓLO de producto_id (¡VIOLACIÓN 2NF!)
- producto_precio_proveedor   --> Depende SÓLO de producto_id (¡VIOLACIÓN 2NF!)
```

Si el nombre del producto cambia, tendríamos que actualizarlo en miles de pedidos antiguos.

### Solución: Descomposición en 2NF

1. Extraer los datos del producto a la tabla independiente `productos(id, nombre, precio_proveedor)`.
2. Mantener en la tabla de detalle únicamente los atributos dependientes de la relación completa: `pedidos_detalle(pedido_id, producto_id, cantidad)`.

---

## 4. Tercera Forma Normal (3NF) - Dependencia Transitiva

Una tabla está en **3NF** si:
1. Ya cumple con la **2NF**.
2. **No existen dependencias transitivas:** Ningún atributo no clave debe depender de otro atributo no clave. Todos los atributos deben depender **directamente y únicamente** de la Clave Primaria.

$$A \rightarrow B \quad \text{y} \quad B \rightarrow C \implies A \rightarrow C \quad \text{(Dependencia Transitiva)}$$

### Ejemplo de Violación de 3NF

Tabla de empleados:
```
id_empleado (PK) -> departamento_id -> departamento_nombre
```
El `departamento_nombre` no depende del empleado, depende del departamento. Si todos los empleados de un departamento renuncian, el departamento desaparece de la base de datos (Anomalía de borrado).

### La Regla Mnemotécnica de Bill Kent

> *"Every non-key attribute must provide a fact about the key, the whole key, and nothing but the key, so help me Codd."*
> 
> - **"The Key"** = 1NF (Identificación por clave).
> - **"The Whole Key"** = 2NF (Dependencia de toda la clave, no de una parte).
> - **"Nothing But the Key"** = 3NF (Dependencia directa sin intermediarios transitivos).

---

## 5. Forma Normal de Boyce-Codd (BCNF)

Es una versión ligeramente más estricta de la 3NF para casos especiales donde una tabla posee múltiples claves candidatas compuestas que se solapan. 

Estipula que para cualquier dependencia funcional $X \rightarrow Y$, **$X$ debe ser obligatoriamente una superclave**. En el 99% de las aplicaciones comerciales, un esquema que alcanza 3NF ya se encuentra en BCNF.

---

## 6. ¿Cuándo y Por Qué Desnormalizar en Producción?

La normalización extrema (más allá de 3NF) no siempre es el objetivo final. Los ingenieros experimentados saben cuándo romper las reglas de forma controlada.

### OLTP vs OLAP

| Criterio | OLTP (Transaccional - Apps Web, APIs) | OLAP (Analítico - Dashboards, BI) |
| :--- | :--- | :--- |
| **Objetivo** | Inserciones, modificaciones y borrados rápidos y seguros. | Consultas de lectura masivas y agregaciones complejas. |
| **Nivel de Normalización** | **3NF Estricto** (Evita bloqueos y redundancias). | **Desnormalizado** (Esquema en Estrella o Copo de Nieve). |
| **Operaciones Comunes** | `INSERT`, `UPDATE` de un solo registro por ID. | `SUM()`, `AVG()` sobre millones de registros históricos. |
| **Prioridad** | Consistencia matemática e integridad ACID. | Velocidad de lectura sin JOINs costosos. |

### Snapshots Históricos Obligatorios (Inmutabilidad Financiera)

Hay un caso donde **duplicar un dato intencionalmente NO es un error de normalización**, sino una regla legal y de negocio:

```sql
-- Al registrar una venta:
CREATE TABLE detalle_ventas (
    venta_id BIGINT,
    producto_id BIGINT,
    cantidad INT,
    -- Guardar el precio cobrado en ESE momento exacto:
    precio_unitario_cobrado DECIMAL(12, 2) NOT NULL,
    PRIMARY KEY (venta_id, producto_id)
);
```
> [!IMPORTANT]
> Si no guardas el `precio_unitario_cobrado` en la tabla de detalle y siempre consultas `productos.precio_actual`, cuando la inflación obligue a subir el precio de un producto el mes que viene, **todas las facturas del año pasado cambiarán de valor**, arruinando la contabilidad legal de la empresa.
