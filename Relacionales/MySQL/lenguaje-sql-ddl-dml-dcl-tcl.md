# Taxonomía del Lenguaje SQL: DDL, DML, DQL, DCL y TCL en MySQL

![Engine](https://img.shields.io/badge/Engine-MySQL%208.0+-4479A1?style=flat&logo=mysql&logoColor=white)
![Focus](https://img.shields.io/badge/Focus-SQL%20Sublanguages%20%26%20Database%20Operations-blue)
![Level](https://img.shields.io/badge/Level-Intermediate%20to%20Senior-orange)

El lenguaje SQL no es un bloque monolítico; está subdividido formalmente en cinco sublenguajes o familias de sentencias según su propósito, su impacto en el almacenamiento físico, su comportamiento transaccional y su nivel de permisos en el motor relacional.

---

## Índice de Contenido

- [1. Mapa General de la Taxonomía SQL](#1-mapa-general-de-la-taxonomía-sql)
- [2. DDL (Data Definition Language) - Definición de Estructura](#2-ddl-data-definition-language---definición-de-estructura)
  - [Sentencias Clave: `CREATE`, `ALTER`, `DROP`, `TRUNCATE`](#sentencias-clave-create-alter-drop-truncate)
  - [El Gran Peligro: Auto-commit Implícito en DDL](#el-gran-peligro-auto-commit-implícito-en-ddl)
  - [Comparativa Técnica: `DROP` vs `TRUNCATE` vs `DELETE`](#comparativa-técnica-drop-vs-truncate-vs-delete)
- [3. DML (Data Manipulation Language) - Manipulación de Datos](#3-dml-data-manipulation-language---manipulación-de-datos)
  - [Operaciones CRUD: `INSERT`, `UPDATE`, `DELETE`](#operaciones-crud-insert-update-delete)
  - [Buenas Prácticas: `SQL_SAFE_UPDATES` y Transacciones Previas](#buenas-prácticas-sql_safe_updates-y-transacciones-previas)
  - [Manejo Avanzado: `INSERT ... ON DUPLICATE KEY UPDATE`](#manejo-avanzado-insert--on-duplicate-key-update)
- [4. DQL (Data Query Language) - Recuperación de Información](#4-dql-data-query-language---recuperación-de-información)
  - [La Sentencia `SELECT` y su Orden Lógico de Ejecución](#la-sentencia-select-y-su-orden-lógico-de-ejecución)
- [5. TCL (Transaction Control Language) - Control Transaccional](#5-tcl-transaction-control-language---control-transaccional)
  - [Fundamento ACID y el Ciclo `START TRANSACTION`, `COMMIT`, `ROLLBACK`](#fundamento-acid-y-el-ciclo-start-transaction-commit-rollback)
  - [Puntos de Restauración: `SAVEPOINT` y `ROLLBACK TO`](#puntos-de-restauración-savepoint-y-rollback-to)
- [6. DCL (Data Control Language) - Seguridad y Privilegios](#6-dcl-data-control-language---seguridad-y-privilegios)
  - [Anatomía del Usuario en MySQL: `usuario@host`](#anatomía-del-usuario-en-mysql-usuariohost)
  - [Concesión y Revocación: `GRANT` y `REVOKE`](#concesión-y-revocación-grant-y-revoke)
  - [Principio de Menor Privilegio en Entornos de Producción](#principio-de-menor-privilegio-en-entornos-de-producción)
- [7. Cuadro Comparativo de Sublenguajes](#7-cuadro-comparativo-de-sublenguajes)

---

## 1. Mapa General de la Taxonomía SQL

| Sublenguaje | Significado | Objetivo Principal | Comandos Representativos | ¿Soporta `ROLLBACK`? |
| :--- | :--- | :--- | :--- | :---: |
| **DDL** | Data Definition Language | Crear, modificar o destruir la estructura de esquemas y objetos. | `CREATE`, `ALTER`, `DROP`, `TRUNCATE` | **No** (Auto-commit) |
| **DML** | Data Manipulation Language | Insertar, modificar y eliminar registros en las tablas. | `INSERT`, `UPDATE`, `DELETE` | **Sí** |
| **DQL** | Data Query Language | Consultar y proyectar datos almacenados sin alterarlos. | `SELECT` | N/A (Solo lectura) |
| **TCL** | Transaction Control Language | Administrar los límites y la atomicidad de las transacciones ACID. | `COMMIT`, `ROLLBACK`, `SAVEPOINT` | N/A (Es el control) |
| **DCL** | Data Control Language | Gestionar credenciales, roles y permisos de acceso al motor. | `CREATE USER`, `GRANT`, `REVOKE` | Motor-dependiente |

---

## 2. DDL (Data Definition Language) - Definición de Estructura

Las sentencias DDL interactúan directamente con el catálogo de metadatos del motor relacional y con la estructura física del almacenamiento (`.ibd` en InnoDB).

### Sentencias Clave

- **`CREATE`**: Instancia nuevos objetos (bases de datos, tablas, vistas, índices, procedimientos almacenados).
- **`ALTER`**: Modifica la definición física o lógica de un objeto existente (añadir columnas, alterar tipos, agregar constraints).
- **`DROP`**: Destruye por completo el objeto y sus metadatos del catálogo.
- **`TRUNCATE`**: Vacía completamente los datos de una tabla reiniciando sus contadores internos.

### El Gran Peligro: Auto-commit Implícito en DDL

> [!WARNING]
> En MySQL (y en la mayoría de los motores relacionales), **toda sentencia DDL provoca un `COMMIT` implícito automático e irrevocable** de cualquier transacción en curso antes y después de su ejecución.
>
> Esto significa que **es imposible hacer `ROLLBACK` de un comando DDL**.

```sql
-- EJEMPLO DEL RIESGO DE AUTO-COMMIT IMPLÍCITO:
START TRANSACTION;

DELETE FROM facturas WHERE id = 105; -- Operación DML

-- Si por descuido ejecutas un DDL dentro de la transacción:
ALTER TABLE clientes ADD COLUMN observaciones VARCHAR(255); 
-- << MySQL ejecuta un COMMIT silencioso aquí >>

ROLLBACK; 
-- El ROLLBACK NO deshará el DELETE anterior, porque el ALTER TABLE lo confirmó permanentemente en disco.
```

### Comparativa Técnica: `DROP` vs `TRUNCATE` vs `DELETE`

| Aspecto | `DELETE FROM tabla;` | `TRUNCATE TABLE tabla;` | `DROP TABLE tabla;` |
| :--- | :--- | :--- | :--- |
| **Familia** | **DML** | **DDL** | **DDL** |
| **Mecánica Física** | Escanea y borra fila por fila, registrando cada eliminación en el log de transacciones (*Undo/Redo log*). | Destruye el archivo de datos/páginas y crea un segmento vacío nuevo. | Elimina los datos y destruye la definición de la tabla en el diccionario. |
| **Velocidad** | Lenta en tablas con millones de filas. | **Ultrarrápida** $O(1)$. | **Instantánea**. |
| **Clave `AUTO_INCREMENT`** | **Mantiene** el valor consecutivo donde iba. | **Reinicia** el contador a `1` (o su valor inicial). | Desaparece la tabla por completo. |
| **Disparadores (Triggers)** | Dispara eventos `ON DELETE`. | **No** dispara ningún trigger. | **No** dispara triggers. |
| **Reversibilidad** | Reversible con `ROLLBACK` dentro de una transacción. | **Irreversible** (Auto-commit implícito). | **Irreversible**. |

---

## 3. DML (Data Manipulation Language) - Manipulación de Datos

DML engloba las operaciones destinadas a poblar, mutar y depurar la información persistida en las tablas.

### Operaciones CRUD

```sql
-- Create: Inserción de registros
INSERT INTO empleados (nombre, puesto, salario)
VALUES ('Lucía Méndez', 'Backend Developer', 2800.00);

-- Update: Modificación de registros existentes
UPDATE empleados 
SET salario = 3100.00 
WHERE id = 12;

-- Delete: Eliminación de registros específicos
DELETE FROM empleados 
WHERE id = 12;
```

### Buenas Prácticas: `SQL_SAFE_UPDATES` y Transacciones Previas

En MySQL Workbench y clientes profesionales suele estar activa la variable `sql_safe_updates = 1`. Esta protección impide ejecutar sentencias `UPDATE` o `DELETE` que no incluyan una cláusula `WHERE` vinculada a una columna con clave primaria o índice.

```sql
-- Habilitar o deshabilitar temporalmente la protección en sesión:
SET sql_safe_updates = 1;
```

> [!TIP]
> **Técnica Profesional de Seguridad:** Antes de ejecutar un `UPDATE` o `DELETE` crítico en producción, ejecuta siempre primero un `SELECT` con el mismo filtro `WHERE` para validar exactamente cuántas y cuáles filas se verán afectadas:
> ```sql
> -- 1. Auditar primero:
> SELECT COUNT(*), id, email FROM usuarios WHERE ultimo_acceso < '2024-01-01';
> 
> -- 2. Si el conteo coincide exactamente con lo esperado, proceder con la mutación:
> DELETE FROM usuarios WHERE ultimo_acceso < '2024-01-01';
> ```

### Manejo Avanzado: `INSERT ... ON DUPLICATE KEY UPDATE` (Upsert)

Patrón nativo de MySQL para insertar un registro o actualizarlo si ya existe una colisión por clave primaria o índice único (`UNIQUE`):

```sql
INSERT INTO inventario (producto_id, stock, ultima_reposicion)
VALUES (45, 10, NOW())
ON DUPLICATE KEY UPDATE 
    stock = stock + VALUES(stock),
    ultima_reposicion = NOW();
```

---

## 4. DQL (Data Query Language) - Recuperación de Información

Aunque históricamente se agrupaba a `SELECT` dentro de DML, en la ingeniería moderna de bases de datos se categoriza de forma independiente como **DQL (Data Query Language)** porque su ejecución es estrictamente de lectura (*Idempotente y libre de efectos colaterales sobre los datos almacenados*).

### La Sentencia `SELECT` y su Orden Lógico de Ejecución

Un error conceptual recurrente es creer que el motor ejecuta la consulta en el mismo orden en que se escribe:

```
Orden Sintáctico (Cómo se escribe):
SELECT -> FROM -> JOIN -> WHERE -> GROUP BY -> HAVING -> ORDER BY -> LIMIT

Orden Lógico de Procesamiento Interno (Cómo lo evalúa MySQL):
1. FROM / JOIN       --> Localiza las tablas y cruza los conjuntos de datos.
2. WHERE             --> Filtra filas individuales a nivel de registro base.
3. GROUP BY          --> Agrupa los registros resultantes en bloques.
4. HAVING            --> Filtra grupos ya agregados (evalúa COUNT, SUM, etc.).
5. SELECT            --> Proyecta las columnas, calcula alias y funciones de ventana.
6. DISTINCT          --> Deduplica el conjunto proyectado si fue solicitado.
7. ORDER BY          --> Ordena el conjunto final procesado.
8. LIMIT / OFFSET    --> Paginación física final de las filas a entregar.
```

---

## 5. TCL (Transaction Control Language) - Control Transaccional

TCL administra los límites de las transacciones garantizando el cumplimiento del estándar **ACID** (Atomicidad, Consistencia, Aislamiento y Durabilidad).

### Ciclo Transaccional Estándar

```sql
-- 1. Iniciar la frontera transaccional explícita
START TRANSACTION;

-- 2. Secuencia de mutaciones críticas
UPDATE cuentas SET saldo = saldo - 500 WHERE id = 1001;
UPDATE cuentas SET saldo = saldo + 500 WHERE id = 2002;

-- 3. Confirmar permanentemente en disco si todo fue correcto
COMMIT;

-- O bien, revertir por completo si ocurrió un error:
-- ROLLBACK;
```

### Puntos de Restauración: `SAVEPOINT`

Los `SAVEPOINT` permiten crear puntos intermedios de control dentro de una transacción larga para realizar reversiones parciales sin descartar todo el trabajo previo:

```sql
START TRANSACTION;

INSERT INTO pedidos (id, cliente_id, total) VALUES (501, 10, 150.00);
SAVEPOINT punto_pedido;

INSERT INTO detalle_pedidos (pedido_id, producto_id, cantidad) VALUES (501, 9999, 1);
-- Si el producto 9999 no existe o falla una validación:
ROLLBACK TO punto_pedido; -- Deshace solo el detalle_pedidos

-- El pedido principal sigue existiendo en la transacción actual:
COMMIT;
```

---

## 6. DCL (Data Control Language) - Seguridad y Privilegios

DCL define la capa de autorización y control de acceso al motor de bases de datos.

### Anatomía del Usuario en MySQL: `usuario@host`

En MySQL, un usuario no es solo un nombre; **el host desde el que se conecta forma parte inseparable de su identidad de seguridad**:

- `'admin'@'localhost'`: Puede conectarse única y exclusivamente desde la misma máquina física donde corre el servidor de MySQL.
- `'app_backend'@'192.168.1.50'`: Puede conectarse únicamente desde esa dirección IP privada fija.
- `'reportero'@'%'`: El comodín `%` permite conexiones desde cualquier dirección IP externa.

```sql
-- Crear un usuario con contraseña cifrada nativa
CREATE USER 'app_ecommerce'@'localhost' IDENTIFIED BY 'PasswordSeguro!2026';
```

### Concesión y Revocación: `GRANT` y `REVOKE`

```sql
-- Conceder permisos DML y DQL sobre una base de datos específica
GRANT SELECT, INSERT, UPDATE, DELETE ON tienda.* TO 'app_ecommerce'@'localhost';

-- Revocar privilegios peligrosos (ej. eliminación)
REVOKE DELETE ON tienda.* FROM 'app_ecommerce'@'localhost';

-- Aplicar cambios de privilegios en el catálogo de memoria de MySQL
FLUSH PRIVILEGES;
```

### Principio de Menor Privilegio en Entornos de Producción

> [!CAUTION]
> **Antipatrón Crítico:** Conectar la aplicación web o API a la base de datos con el usuario `root` o asignando `GRANT ALL PRIVILEGES`.
>
> Si la aplicación sufre una vulnerabilidad de Inyección SQL (SQLi), un atacante con permisos `ALL` o `SUPER` podrá ejecutar `DROP DATABASE`, alterar usuarios del sistema o leer archivos confidenciales del sistema operativo mediante `LOAD DATA INFILE`.
>
> **Regla de Producción:** El usuario de la aplicación debe tener únicamente permisos DML y DQL estrictos (`SELECT`, `INSERT`, `UPDATE`, `DELETE`) y carecer por completo de permisos DDL (`DROP`, `ALTER`).

---

## 7. Cuadro Comparativo de Sublenguajes

| Dimensión | DDL | DML | DQL | TCL | DCL |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Sujeto de acción** | Metadatos y Estructura | Registros y Tuplas | Conjunto de Resultados | Transacciones | Cuentas y Privilegios |
| **Rol habitual** | DBA / Arquitecto / Migraciones | Aplicación / API | Aplicación / Reportes / BI | Motor / Backend | SysAdmin / DBA |
| **Riesgo Operativo** | Crítico (Estructural) | Alto (Mutación de datos) | Bajo (Consumo de RAM/CPU) | Medio (Bloqueos) | Crítico (Seguridad) |
| **Impacto Transaccional** | Fuerza `COMMIT` | Respeta la transacción | No genera mutaciones | Define los límites | Actualiza catálogo de acceso |
