# Vistas, Vistas Materializadas y Triggers (Disparadores) en MySQL

![Engine](https://img.shields.io/badge/Engine-MySQL%208.0+-4479A1?style=flat&logo=mysql&logoColor=white)
![Focus](https://img.shields.io/badge/Focus-Views%2C%20Materialized%20Patterns%20%26%20Triggers-blue)
![Level](https://img.shields.io/badge/Level-Intermediate%20to%20Senior-orange)

Guía técnica sobre la abstracción lógica de datos, seguridad por proyección y automatización de eventos en el motor relacional. Cubre la creación de Vistas estándar, algoritmos de procesamiento (`MERGE` vs `TEMPTABLE`), vistas actualizables con `WITH CHECK OPTION`, la comparativa arquitectónica entre **Vistas vs CTEs**, patrones de implementación de Vistas Materializadas en MySQL, y el diseño riguroso de Triggers (`BEFORE`/`AFTER`) junto con sus riesgos de rendimiento en producción.

---

## Índice de Contenido

- [1. Vistas Estándar (Views)](#1-vistas-estándar-views)
  - [¿Qué es una Vista y qué problema resuelve?](#qué-es-una-vista-y-qué-problema-resuelve)
  - [Sintaxis y Creación (`CREATE OR REPLACE VIEW`)](#sintaxis-y-creación-create-or-replace-view)
  - [Seguridad y Abstracción: Ocultamiento de Columnas Sensibles](#seguridad-y-abstracción-ocultamiento-de-columnas-sensibles)
  - [Algoritmos de Procesamiento: `MERGE` vs `TEMPTABLE`](#algoritmos-de-procesamiento-merge-vs-temptable)
  - [Vistas Actualizables y la Cláusula Crítica `WITH CHECK OPTION`](#vistas-actualizables-y-la-cláusula-crítica-with-check-option)
- [2. Vista vs CTE (Common Table Expression): Análisis Comparativo](#2-vista-vs-cte-common-table-expression-análisis-comparativo)
  - [Matriz de Diferencias Técnicas](#matriz-de-diferencias-técnicas)
  - [Criterios de Decisión Arquitectónica (¿Cuándo usar cuál?)](#criterios-de-decisión-arquitectónica-cuándo-usar-cuál)
- [3. Vistas Materializadas (Materialized Views)](#3-vistas-materializadas-materialized-views)
  - [La Realidad de MySQL: Falta de Soporte Nativo](#la-realidad-de-mysql-falta-de-soporte-nativo)
  - [Patrón de Emulación en Producción: Tablas de Resumen + Eventos Programados](#patrón-de-emulación-en-producción-tablas-de-resumen--eventos-programados)
- [4. Triggers (Disparadores)](#4-triggers-disparadores)
  - [Concepto y Momentos de Disparo: `BEFORE` vs `AFTER`](#concepto-y-momentos-de-disparo-before-vs-after)
  - [Los Pseudorregistros `NEW` y `OLD`](#los-pseudorregistros-new-y-old)
  - [Auditoría de Cambios Históricos (Audit Trail Pattern)](#auditoría-de-cambios-históricos-audit-trail-pattern)
  - [Validaciones de Dominio y Modificación al Vuelo con `BEFORE INSERT`](#validaciones-de-dominio-y-modificación-al-vuelo-con-before-insert)
- [5. Peligros y Antipatrones de los Triggers en Producción](#5-peligros-y-antipatrones-de-los-triggers-en-producción)
  - [Efectos Secundarios Invisibles y Dificultad de Depuración](#efectos-secundarios-invisibles-y-dificultad-de-depuración)
  - [Degradación Crítica en Inserciones Masivas (Bulk Inserts)](#degradación-crítica-en-inserciones-masivas-bulk-inserts)
- [6. Resumen de Directrices de Ingeniería](#6-resumen-de-directrices-de-ingeniería)

---

## 1. Vistas Estándar (Views)

Una **Vista** es una tabla virtual cuyo contenido no está almacenado físicamente en disco; se define a partir de una consulta SQL predefinida que se ejecuta dinámicamente cada vez que la vista es consultada.

### ¿Qué es una Vista y qué problema resuelve?

1. **Simplificación de Consultas:** Encapsula consultas complejas con múltiples `JOINs`, agregaciones y funciones de formato en una sola entidad accesible con un `SELECT * FROM mi_vista;`.
2. **Capa de Abstracción e Independencia Lógica:** Si el esquema de las tablas base cambia (por ejemplo, se renombran columnas o se normalizan tablas), la vista puede adaptarse internamente para que las aplicaciones externas y reportes sigan consumiendo los mismos nombres de columnas sin romperse.
3. **Seguridad y Control de Acceso:** Permite otorgar permisos de lectura a usuarios o roles sobre la vista sin darles acceso a las tablas físicas subyacentes.

### Sintaxis y Creación

```sql
CREATE OR REPLACE VIEW v_resumen_pedidos AS
SELECT 
    p.id AS pedido_id,
    c.nombre AS cliente_nombre,
    c.email AS cliente_email,
    p.fecha_pedido,
    p.total,
    p.estado
FROM pedidos p
INNER JOIN clientes c ON p.cliente_id = c.id;
```

---

### Seguridad y Abstracción: Ocultamiento de Columnas Sensibles

Imagina una tabla de `empleados` que contiene salarios, cuentas bancarias y contraseñas. Un departamento de marketing o RRHH operativo necesita ver la lista de empleados pero no sus datos confidenciales.

```sql
-- 1. Crear una vista filtrada:
CREATE OR REPLACE VIEW v_empleados_publico AS
SELECT 
    id,
    nombre,
    apellido,
    departamento,
    email,
    fecha_ingreso
FROM empleados;

-- 2. Otorgar permisos EXCLUSIVAMENTE a la vista:
GRANT SELECT ON empresa.v_empleados_publico TO 'usuario_reportes'@'localhost';
-- El usuario no tiene permiso sobre la tabla 'empleados' y no podrá ver el salario.
```

---

### Algoritmos de Procesamiento: `MERGE` vs `TEMPTABLE`

MySQL ofrece tres algoritmos de procesamiento para vistas (`ALGORITHM = {UNDEFINED | MERGE | TEMPTABLE}`):

1. **`MERGE` (Óptimo y Preferido):**
   - MySQL combina el texto de la consulta externa con la definición de la vista en una sola sentencia SQL optimizada.
   - Permite que los índices de las tablas base se aprovechen completamente para los filtros `WHERE` de la consulta externa.
2. **`TEMPTABLE`:**
   - MySQL ejecuta la consulta de la vista, guarda el resultado completo en una **tabla temporal en memoria o disco**, y luego ejecuta la consulta externa sobre esa tabla temporal.
   - **Desventaja:** Pérdida de índices originales y mayor consumo de memoria RAM.
   - **Cuándo se fuerza automáticamente:** Cuando la vista incluye `GROUP BY`, `DISTINCT`, funciones de agregación (`SUM`, `COUNT`), `UNION`, o subconsultas en el `SELECT`.

```sql
-- Especificar algoritmo explícito:
CREATE ALGORITHM = MERGE VIEW v_clientes_activos AS
SELECT id, nombre, email 
FROM clientes 
WHERE activo = 1;
```

---

### Vistas Actualizables y la Cláusula Crítica `WITH CHECK OPTION`

Bajo ciertas condiciones (vistas simples basadas en una sola tabla sin agrupaciones), MySQL permite ejecutar `INSERT`, `UPDATE` o `DELETE` **directamente sobre la vista**, propagando los cambios a la tabla base.

#### El Riesgo de Modificaciones Creadoras de Datos Incoherentes

Si tienes una vista que solo muestra clientes de Bogotá:

```sql
CREATE VIEW v_clientes_bogota AS
SELECT id, nombre, ciudad 
FROM clientes 
WHERE ciudad = 'Bogotá';
```

Si alguien ejecuta:
```sql
UPDATE v_clientes_bogota SET ciudad = 'Medellín' WHERE id = 10;
```
La sentencia tendrá éxito, pero **el registro desaparecerá inmediatamente de la vista** (porque ya no cumple el `WHERE`), provocando confusión y posibles anomalías de datos.

#### La Solución Obligatoria: `WITH CHECK OPTION`

```sql
CREATE OR REPLACE VIEW v_clientes_bogota AS
SELECT id, nombre, ciudad 
FROM clientes 
WHERE ciudad = 'Bogotá'
WITH CHECK OPTION; -- << Bloquea inserciones o actualizaciones incompatibles
```

> [!IMPORTANT]
> Con `WITH CHECK OPTION`, si un usuario intenta insertar o modificar una fila a través de la vista asignándole una ciudad distinta a `'Bogotá'`, MySQL **abortará la operación con un error de violación de constraint** (`CHECK OPTION failed`).

---

## 2. Vista vs CTE (Common Table Expression): Análisis Comparativo

Uno de los dilemas más comunes en el desarrollo SQL es cuándo utilizar una **Vista** y cuándo un **CTE (`WITH ... AS`)**.

### Matriz de Diferencias Técnicas

| Dimensión | Vista (`CREATE VIEW`) | CTE (`WITH ... AS`) |
| :--- | :--- | :--- |
| **Persistencia en Catálogo** | **Permanente:** Se guarda en el diccionario de metadatos de MySQL. | **Efímera / Volátil:** No se almacena; existe solo durante la consulta. |
| **Alcance (*Scope*)** | **Global:** Accesible por cualquier sesión, aplicación o reporte con permisos. | **Local:** Privada y exclusiva de la sentencia SQL que la declara. |
| **Seguridad y Permisos** | **Alta:** Permite dar permisos granulares (`GRANT SELECT ON vista`) ocultando la tabla base. | **Nula:** Requiere que el usuario tenga acceso directo a las tablas base. |
| **Soporte de Recursividad** | No directamente en la vista. | **Sí** (`WITH RECURSIVE`), ideal para árboles jerárquicos o grafos. |
| **Modificación del Esquema** | Requiere permisos de DDL (`CREATE VIEW`, `ALTER VIEW`). | No requiere permisos de DDL; opera como consulta pura. |
| **Optimización de Ejecución** | Reutilizable de forma consistente por el optimizador de costos. | Diseñada para modularizar una consulta analítica individual. |

### Criterios de Decisión Arquitectónica (¿Cuándo usar cuál?)

#### Usa un **CTE** cuando:
- Estás escribiendo una consulta analítica compleja que utiliza **Funciones de Ventana** (ej. calcular el *Top 3* productos por categoría).
- Quieres hacer tu código SQL legible y modular de arriba hacia abajo para una sola tarea, sin crear objetos permanentes en el servidor.
- Necesitas resolver problemas de jerarquías recursivas (ej. organigrama de jefes/empleados o categorías padre/hijo).

#### Usa una **Vista** cuando:
- La misma lógica de `JOINs` o cálculos de negocio se reutiliza en **múltiples partes de la aplicación**, reportes o herramientas de BI (PowerBI, Tableau, Metabase).
- Quieres implementar una capa de **seguridad por omisión de columnas** (restringir acceso a salarios, tokens o contraseñas).
- Quieres desacoplar el frontend/backend del esquema físico interno de la base de datos.

---

## 3. Vistas Materializadas (Materialized Views)

Una **Vista Materializada** es una vista cuyo resultado **SÍ se calcula y se almacena físicamente en disco**. A diferencia de una vista estándar (que calcula los datos en tiempo real al consultarla), la vista materializada responde de forma instantánea leyendo una copia precalculada.

### La Realidad de MySQL: Falta de Soporte Nativo

> [!NOTE]
> Motores como PostgreSQL u Oracle poseen soporte nativo para vistas materializadas mediante comandos como `CREATE MATERIALIZED VIEW` y `REFRESH MATERIALIZED VIEW`. 
> 
> **MySQL 8.0 NO dispone de vistas materializadas nativas.**

### Patrón de Emulación en Producción: Tablas de Resumen + Eventos

En entornos de alta concurrencia con millones de filas, consultar una vista estándar con agregaciones pesadas (`SUM`, `AVG`, múltiples `JOINs`) congela el servidor. 

La solución profesional en MySQL consiste en emular la vista materializada mediante una **Tabla de Resumen Física** refrescada periódicamente mediante un **MySQL Event (`EVENT SCHEDULER`)**:

```sql
-- 1. Tabla física que almacenará los datos precalculados
CREATE TABLE mv_ventas_mensuales (
    ano_mes VARCHAR(7) NOT NULL PRIMARY KEY,
    total_ventas DECIMAL(14, 2) NOT NULL,
    total_pedidos INT NOT NULL,
    ultima_actualizacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- 2. Habilitar el planificador de eventos de MySQL
SET GLOBAL event_scheduler = ON;

-- 3. Evento programado que refresca la tabla cada noche a las 02:00 AM
DELIMITER //

CREATE EVENT evt_refrescar_mv_ventas
ON SCHEDULE EVERY 1 DAY
STARTS (TIMESTAMP(CURRENT_DATE) + INTERVAL 1 DAY + INTERVAL 2 HOUR)
DO
BEGIN
    -- Vaciado y repoblado seguro
    TRUNCATE TABLE mv_ventas_mensuales;
    
    INSERT INTO mv_ventas_mensuales (ano_mes, total_ventas, total_pedidos)
    SELECT 
        DATE_FORMAT(fecha_pedido, '%Y-%m') AS ano_mes,
        SUM(total) AS total_ventas,
        COUNT(id) AS total_pedidos
    FROM pedidos
    GROUP BY DATE_FORMAT(fecha_pedido, '%Y-%m');
END //

DELIMITER ;
```

---

## 4. Triggers (Disparadores)

Un **Trigger** es un objeto de base de datos asociado a una tabla que se ejecuta automáticamente (se "dispara") en respuesta a un evento DML (`INSERT`, `UPDATE` o `DELETE`).

### Concepto y Momentos de Disparo

Cada evento soporta dos momentos de disparo:

| Momento | Descripción | Caso de Uso Típico |
| :--- | :--- | :--- |
| **`BEFORE`** | Se ejecuta **antes** de que el cambio se escriba en la tabla o en el log de transacciones. | Validación defensiva de datos, formateo/limpieza al vuelo o cálculo de valores por defecto. |
| **`AFTER`** | Se ejecuta **después** de que el cambio fue aplicado con éxito en la tabla. | Auditoría histórica, replicación interna a tablas de log o sincronización de estados. |

---

### Los Pseudorregistros `NEW` y `OLD`

Dentro del cuerpo de un trigger, MySQL proporciona acceso a dos variables especiales de registro:

- **`NEW`**: Representa los datos de la fila que se está insertando o el nuevo estado modificado.
- **`OLD`**: Representa el estado previo de la fila antes de ser modificada o eliminada.

| Operación DML | `OLD` disponible | `NEW` disponible |
| :---: | :---: | :---: |
| **`INSERT`** | No (no había fila previa) | **Sí** |
| **`UPDATE`** | **Sí** (valores originales) | **Sí** (nuevos valores) |
| **`DELETE`** | **Sí** (valores eliminados) | No (la fila ya no existe) |

---

### Auditoría de Cambios Históricos (Audit Trail Pattern)

Caso de uso estándar: Registrar en una tabla independiente quién modificó el salario de un empleado, cuál era el valor anterior, cuál es el nuevo valor y en qué fecha ocurrió:

```sql
-- 1. Tabla de auditoría
CREATE TABLE auditoria_empleados (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    empleado_id BIGINT NOT NULL,
    salario_anterior DECIMAL(12, 2) NOT NULL,
    salario_nuevo DECIMAL(12, 2) NOT NULL,
    usuario_modificacion VARCHAR(100) NOT NULL,
    fecha_modificacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- 2. Trigger AFTER UPDATE
DELIMITER //

CREATE TRIGGER trg_auditar_cambio_salario
AFTER UPDATE ON empleados
FOR EACH ROW
BEGIN
    -- Se dispara solo si el salario realmente cambió
    IF OLD.salario <> NEW.salario THEN
        INSERT INTO auditoria_empleados (
            empleado_id, 
            salario_anterior, 
            salario_nuevo, 
            usuario_modificacion
        ) VALUES (
            OLD.id, 
            OLD.salario, 
            NEW.salario, 
            CURRENT_USER()
        );
    END IF;
END //

DELIMITER ;
```

---

### Validaciones de Dominio y Modificación al Vuelo con `BEFORE INSERT`

Un trigger `BEFORE` permite alterar el valor de `NEW` antes de que toque el disco:

```sql
DELIMITER //

CREATE TRIGGER trg_limpiar_datos_cliente
BEFORE INSERT ON clientes
FOR EACH ROW
BEGIN
    -- Forzar email a minúsculas y limpiar espacios accidentales
    SET NEW.email = LOWER(TRIM(NEW.email));
    
    -- Si no se proveyó estado, asignar valor por defecto
    IF NEW.estado IS NULL THEN
        SET NEW.estado = 'ACTIVO';
    END IF;
END //

DELIMITER ;
```

---

## 5. Peligros y Antipatrones de los Triggers en Producción

A pesar de su utilidad, los triggers son una de las herramientas más controvertidas en la ingeniería de bases de datos moderna.

### Efectos Secundarios Invisibles y Dificultad de Depuración

1. **Lógica Oculta:** Cuando un desarrollador de backend ejecuta un simple `UPDATE usuarios SET activo = 0;`, espera que se actualice una columna. Si un trigger oculto en la BD desencadena actualizaciones en cascada, bloqueos de tablas o envíos a tablas de auditoría, el comportamiento del sistema se vuelve impredecible y muy difícil de depurar desde el código de la aplicación.
2. **Imposibilidad de Pruebas Unitarias Aisladas:** La lógica de negocio queda dispersa entre el código del repositorio (Node, Python, Java, etc.) y la base de datos.

### Degradación Crítica en Inserciones Masivas (Bulk Inserts)

> [!CAUTION]
> Los triggers en MySQL operan **estrictamente por fila (`FOR EACH ROW`)**. No existe el concepto de "Trigger por sentencia" (*Statement-level trigger*).
>
> Si ejecutas una carga masiva de datos (*Bulk Insert*) de **500.000 registros**, el trigger se ejecutará **500.000 veces secuencialmente**. Cada ejecución añade sobrecarga de CPU, escrituras adicionales y transacciones internas, multiplicando el tiempo de carga por 10 o 50 veces.

---

## 6. Resumen de Directrices de Ingeniería

1. **Usa Vistas para Lectura y Abstracción:** Ideal para ocultar datos sensibles y unificar reportes comunes.
2. **Aplica siempre `WITH CHECK OPTION` en vistas actualizables:** Previene inserciones o mutaciones que violen los filtros de la propia vista.
3. **Usa CTEs para consultas analíticas locales:** Evita poblar el catálogo de metadatos con vistas de un solo uso.
4. **Reserva los Triggers para Auditoría e Integridad Estricta:** Evita implementar lógica compleja de negocio dentro de un trigger; esa responsabilidad corresponde a la capa de servicios del backend.
5. **Cuidado con las Cargas Masivas:** Si necesitas realizar una migración o inserción de millones de filas en una tabla con triggers, evalúa deshabilitar temporalmente los triggers o realizar la auditoría por lotes en la aplicación.
