# Procedimientos Almacenados, Transacciones ACID y Control de Concurrencia en MySQL

![Engine](https://img.shields.io/badge/Engine-MySQL%208.0+-4479A1?style=flat&logo=mysql&logoColor=white)
![Focus](https://img.shields.io/badge/Focus-Stored%20Procedures%20%26%20ACID%20Transactions-blue)
![Level](https://img.shields.io/badge/Level-Senior%20Engineering-orange)

Guía técnica avanzada sobre la implementación de lógica de negocio transaccional dentro del motor relacional. Cubre la creación de Procedimientos Almacenados (`STORED PROCEDURES`), manejo robusto de excepciones (`HANDLERS`), propiedades **ACID** en InnoDB (*Undo Log* y *Redo Log*), niveles de aislamiento (*Isolation Levels*), estrategias de bloqueo pesimista (`FOR UPDATE`) y prevención de interbloqueos (*Deadlocks*).

---

## Índice de Contenido

- [1. Procedimientos Almacenados (Stored Procedures)](#1-procedimientos-almacenados-stored-procedures)
  - [¿Por qué ejecutar lógica en el motor? (Pros y Contras en Arquitectura)](#por-qué-ejecutar-lógica-en-el-motor-pros-y-contras-en-arquitectura)
  - [El Delimitador de Comandos (`DELIMITER`)](#el-delimitador-de-comandos-delimiter)
  - [Parámetros: `IN`, `OUT` e `INOUT`](#parámetros-in-out-e-inout)
  - [Variables Locales (`DECLARE`), Asignación y Estructuras de Control (`IF`, `CASE`)](#variables-locales-declare-asignación-y-estructuras-de-control-if-case)
  - [Manejo de Errores Profesional (`DECLARE ... HANDLER FOR SQLEXCEPTION`)](#manejo-de-errores-profesional-declare--handler-for-sqlexception)
- [2. Transacciones ACID Bajo el Capó (Mecánica de InnoDB)](#2-transacciones-acid-bajo-el-capó-mecánica-de-innodb)
  - [Desglose Físico de ACID: Atomicidad, Consistencia, Aislamiento y Durabilidad](#desglose-físico-de-acid-atomicidad-consistencia-aislamiento-y-durabilidad)
  - [El Rol del Undo Log (Reversión y MVCC) vs Redo Log (Crash Recovery)](#el-rol-del-undo-log-reversión-y-mvcc-vs-redo-log-crash-recovery)
  - [La Trampa de `autocommit = 1`](#la-trampa-de-autocommit--1)
- [3. Niveles de Aislamiento de Transacciones (Isolation Levels)](#3-niveles-de-aislamiento-de-transacciones-isolation-levels)
  - [Los Tres Fenómenos de Concurrencia (Lectura Sucia, No Repetible y Fantasma)](#los-tres-fenómenos-de-concurrencia-lectura-sucia-no-repetible-y-fantasma)
  - [Matriz de Niveles de Aislamiento en MySQL](#matriz-de-niveles-de-aislamiento-en-mysql)
  - [Comportamiento por Defecto en InnoDB: `REPEATABLE READ`](#comportamiento-por-defecto-en-innodb-repeatable-read)
- [4. Bloqueos (Locking) y Concurrencia Pesimista](#4-bloqueos-locking-y-concurrencia-pesimista)
  - [Bloqueo Compartido (`FOR SHARE` / `LOCK IN SHARE MODE`)](#bloqueo-compartido-for-share--lock-in-share-mode)
  - [Bloqueo Exclusivo (`FOR UPDATE`)](#bloqueo-exclusivo-for-update)
  - [Deadlocks (Interbloqueos Mortales): Causa y Resolución Automática en InnoDB](#deadlocks-interbloqueos-mortales-causa-y-resolución-automática-en-innodb)
- [5. Patrón de Oro: Procedimiento Transaccional Bancario con Manejo de Fallos](#5-patrón-de-oro-procedimiento-transaccional-bancario-con-manejo-de-fallos)
- [6. Resumen de Directrices de Producción](#6-resumen-de-directrices-de-producción)

---

## 1. Procedimientos Almacenados (Stored Procedures)

Un **Procedimiento Almacenado** es un bloque de sentencias SQL compilado y persistido en el catálogo de la base de datos que puede aceptar parámetros, ejecutar lógica procedural (bucles, condicionales), manipular múltiples tablas y devolver conjuntos de resultados o valores de salida.

### ¿Por qué ejecutar lógica en el motor? (Pros y Contras en Arquitectura)

| Ventajas | Desventajas |
| :--- | :--- |
| **Menor latencia de red:** Reduce múltiples viajes de ida y vuelta (*network roundtrips*) entre la app y la base de datos a una sola invocación `CALL`. | **Consumo de CPU en el servidor de base de datos:** El escalado vertical de la BD es mucho más costoso que el escalado horizontal de servidores de backend. |
| **Seguridad por encapsulación:** Permite revocar permisos DML directos sobre las tablas base a la aplicación y permitir solo `EXECUTE` sobre procedimientos autorizados. | **Dificultad de pruebas y CI/CD:** El control de versiones, despliegue continuo y tests unitarios son más complejos que en el código de backend. |
| **Atomicidad garantizada cerca de los datos:** Toda la operación transaccional se gestiona internamente en la memoria del motor. | **Acoplamiento al dialecto:** Migrar de MySQL a PostgreSQL u Oracle requiere reescribir toda la lógica procedural. |

---

### El Delimitador de Comandos (`DELIMITER`)

En SQL estándar, el carácter punto y coma (`;`) indica el fin de una sentencia ejecutable. Sin embargo, dentro de un procedimiento almacenado se escriben múltiples sentencias intermedias, cada una finalizada con `;`. 

Para evitar que el cliente de MySQL intente ejecutar el procedimiento a mitad de camino antes de llegar al `END`, se cambia temporalmente el delimitador global:

```sql
DELIMITER //

CREATE PROCEDURE mi_procedimiento()
BEGIN
    SELECT NOW();
    SELECT DATABASE();
END //

DELIMITER ;
```

---

### Parámetros: `IN`, `OUT` e `INOUT`

MySQL soporta tres modos de parámetros:

1. **`IN` (Por defecto):** Parámetro de entrada. El procedimiento recibe una copia del valor proporcionado por quien lo invoca; modificarlo internamente no altera el valor exterior.
2. **`OUT`:** Parámetro de salida. Se pasa una variable para que el procedimiento le asigne un valor que podrá ser leído por la aplicación tras la finalización.
3. **`INOUT`:** Parámetro bidireccional. Entra con un valor inicial, el procedimiento lo procesa y lo sobrescribe con un nuevo resultado.

```sql
DELIMITER //

CREATE PROCEDURE calcular_estadisticas_ventas(
    IN p_cliente_id BIGINT,
    OUT p_total_gastado DECIMAL(12, 2),
    OUT p_cantidad_pedidos INT
)
BEGIN
    SELECT 
        COALESCE(SUM(total), 0.00),
        COUNT(*)
    INTO 
        p_total_gastado,
        p_cantidad_pedidos
    FROM pedidos
    WHERE cliente_id = p_cliente_id;
END //

DELIMITER ;

-- Invocación y lectura de parámetros de salida:
CALL calcular_estadisticas_ventas(15, @gasto, @pedidos);
SELECT @gasto AS total_facturado, @pedidos AS total_compras;
```

---

### Variables Locales (`DECLARE`), Asignación y Estructuras de Control

Las variables locales deben declararse al inicio estricto del bloque `BEGIN ... END` antes de cualquier sentencia DML:

```sql
DELIMITER //

CREATE PROCEDURE evaluar_categoria_cliente(
    IN p_cliente_id BIGINT,
    OUT p_categoria VARCHAR(20)
)
BEGIN
    -- 1. Declaración de variables locales
    DECLARE v_total_compras DECIMAL(12, 2) DEFAULT 0.00;

    -- 2. Asignación mediante SELECT INTO
    SELECT COALESCE(SUM(total), 0)
    INTO v_total_compras
    FROM pedidos
    WHERE cliente_id = p_cliente_id;

    -- 3. Control de flujo condicional
    IF v_total_compras >= 5000.00 THEN
        SET p_categoria = 'PLATINUM';
    ELSEIF v_total_compras >= 1000.00 THEN
        SET p_categoria = 'GOLD';
    ELSE
        SET p_categoria = 'STANDARD';
    END IF;
END //

DELIMITER ;
```

---

### Manejo de Errores Profesional (`DECLARE ... HANDLER FOR SQLEXCEPTION`)

> [!CAUTION]
> Un procedimiento que ejecuta transacciones sin un manejador de excepciones es un riesgo crítico. Si una sentencia falla por violación de clave foránea o duplicidad, la transacción puede quedar abierta consumiendo bloqueos en la base de datos de forma indefinida.

Un **Handler** permite atrapar cualquier error de SQL (`SQLEXCEPTION`) o advertencia (`SQLWARNING`) para ejecutar una rutina de contingencia (como un `ROLLBACK` forzado) y retornar un código de fallo limpio:

```sql
DECLARE EXIT HANDLER FOR SQLEXCEPTION
BEGIN
    -- Código de recuperación ante cualquier error SQL:
    ROLLBACK;
    SET p_codigo_respuesta = -1;
    SET p_mensaje_respuesta = 'Error interno en la ejecución de la transacción.';
END;
```

---

## 2. Transacciones ACID Bajo el Capó (Mecánica de InnoDB)

Una **Transacción** es una unidad lógica de trabajo que agrupa una o más sentencias SQL ejecutadas de forma indivisible.

### Desglose Físico de ACID

```
A - Atomicidad:   O se confirman todas las operaciones o no se aplica ninguna.
C - Consistencia: El esquema pasa de un estado válido a otro sin violar restricciones.
I - Aislamiento:  Las transacciones concurrentes no interfieren entre sí de forma imprevista.
D - Durabilidad:  Una vez confirmado (COMMIT), el dato sobrevive a fallos eléctricos o del SO.
```

### El Rol del Undo Log vs Redo Log

En el motor de almacenamiento InnoDB:

1. **Undo Log (Memoria de Deshacer):**
   - Almacena las versiones anteriores de las filas antes de ser modificadas.
   - Si se ejecuta un `ROLLBACK`, InnoDB lee el *Undo Log* para restaurar los valores exactos que tenían antes de que la transacción iniciara.
   - Es la base de **MVCC (Multi-Version Concurrency Control)**: permite que las lecturas no bloqueen a las escrituras y que las escrituras no bloqueen a las lecturas.
2. **Redo Log (Memoria de Rehacer / Write-Ahead Logging - WAL):**
   - Antes de escribir las páginas de datos en el archivo físico en disco (`.ibd`), los cambios se escriben secuencialmente en el *Redo Log*.
   - Si el servidor sufre un corte eléctrico instantáneo, al reiniciar el servicio InnoDB lee el *Redo Log* y completa las transacciones que ya habían sido confirmadas con `COMMIT`, garantizando la **Durabilidad**.

### La Trampa de `autocommit = 1`

Por defecto, MySQL opera con la variable de sistema `autocommit = 1`. Esto significa que cada sentencia individual `INSERT`, `UPDATE` o `DELETE` que se ejecute fuera de un bloque transaccional explícito se confirma automáticamente como una transacción irreversible de una sola línea.

Para iniciar una transacción multilínea protegida, siempre debe usarse:

```sql
START TRANSACTION; -- Desactiva temporalmente el autocommit para esta sesión
-- ... Sentencias DML ...
COMMIT; -- O ROLLBACK;
```

---

## 3. Niveles de Aislamiento de Transacciones (Isolation Levels)

Cuando cientos de usuarios interactúan concurrentemente con la base de datos, pueden ocurrir tres anomalías clásicas de concurrencia:

### Fenómenos de Concurrencia

1. **Lectura Sucia (Dirty Read):** La Transacción A lee datos modificados por la Transacción B que **aún no han sido confirmados** (`COMMIT`). Si B hace `ROLLBACK`, A leyó datos fantasmas que nunca existieron formalmente.
2. **Lectura No Repetible (Non-Repeatable Read):** La Transacción A lee una fila. La Transacción B modifica esa misma fila y hace `COMMIT`. Si A vuelve a leer la fila, obtiene valores distintos dentro de la misma transacción.
3. **Lectura Fantasma (Phantom Read):** La Transacción A consulta un rango de filas con una condición `WHERE`. La Transacción B inserta una nueva fila que cumple esa condición y hace `COMMIT`. Si A repite la consulta, aparece una nueva fila "fantasma".

### Matriz de Niveles de Aislamiento en MySQL

| Nivel de Aislamiento | Lectura Sucia | Lectura No Repetible | Lectura Fantasma | Rendimiento Concurrente |
| :--- | :---: | :---: | :---: | :---: |
| **READ UNCOMMITTED** | Permite | Permite | Permite | Máximo (Peligroso) |
| **READ COMMITTED** | Previene | Permite | Permite | Alto |
| **REPEATABLE READ** *(Por defecto en MySQL)* | Previene | Previene | Previene (vía Next-Key Locks) | Balance Óptimo |
| **SERIALIZABLE** | Previene | Previene | Previene | Mínimo (Bloqueo Total) |

> [!NOTE]
> A diferencia de otros motores relacionales como PostgreSQL u Oracle (cuyo nivel por defecto es `READ COMMITTED`), **MySQL InnoDB utiliza `REPEATABLE READ` por defecto**, logrando evitar lecturas fantasmas gracias a su algoritmo de indexación de intervalos (*Next-Key Locking*).

---

## 4. Bloqueos (Locking) y Concurrencia Pesimista

Cuando dos procesos intentan actualizar el mismo registro al mismo tiempo, se requiere control de concurrencia pesimista para evitar condiciones de carrera (*Race Conditions*).

### Bloqueo Compartido (`FOR SHARE`)

Permite que otras sesiones lean la fila, pero les prohíbe actualizarla o eliminarla hasta que la transacción actual finalice.

```sql
SELECT * FROM productos WHERE id = 100 FOR SHARE;
```

### Bloqueo Exclusivo (`FOR UPDATE`)

Adquiere un bloqueo exclusivo sobre las filas leídas. Ninguna otra sesión puede leer con bloqueo ni modificar esas filas hasta que la transacción haga `COMMIT` o `ROLLBACK`.

```sql
-- Caso de Uso Crítico: Descontar saldo o reservar stock
START TRANSACTION;

SELECT stock 
FROM productos 
WHERE id = 50 
FOR UPDATE; -- << Bloquea la fila 50 para que nadie más la modifique

UPDATE productos 
SET stock = stock - 1 
WHERE id = 50;

COMMIT;
```

### Deadlocks (Interbloqueos Mortales)

Un **Deadlock** ocurre cuando dos o más transacciones mantienen bloqueos sobre filas distintas y cada una intenta adquirir un bloqueo exclusivo sobre la fila que la otra mantiene:

```
Sesión 1: Bloquea Fila A -> Intenta acceder a Fila B (Espera a Sesión 2)
Sesión 2: Bloquea Fila B -> Intenta acceder a Fila A (Espera a Sesión 1)
```

Ambas sesiones quedarían congeladas para siempre. 

**¿Cómo lo resuelve InnoDB?**  
InnoDB posee un detector automático de interbloqueos (*Deadlock Detection*). Al identificar el ciclo, **aborta y hace `ROLLBACK` automáticamente a la transacción más pequeña** (la que haya insertado o alterado menos filas), permitiendo que la otra continúe y lanzando el error `1213 (40001): Deadlock found when trying to get lock; try restarting transaction`.

---

## 5. Patrón de Oro: Procedimiento Transaccional Bancario con Manejo de Fallos

A continuación se presenta la arquitectura profesional de un procedimiento almacenado transaccional que realiza una transferencia monetaria entre dos cuentas bancarias:
- Aplica validaciones de saldo.
- Bloquea exclusivamente las filas con `FOR UPDATE`.
- Utiliza un `EXIT HANDLER` para asegurar `ROLLBACK` automático si ocurre un error inesperado de base de datos.
- Devuelve parámetros de estado hacia el backend.

```sql
DELIMITER //

CREATE PROCEDURE sp_transferir_fondos(
    IN p_cuenta_origen BIGINT,
    IN p_cuenta_destino BIGINT,
    IN p_monto DECIMAL(12, 2),
    OUT p_resultado INT,
    OUT p_mensaje VARCHAR(255)
)
sp_bloque: BEGIN
    -- 1. Declaración de variables internas
    DECLARE v_saldo_origen DECIMAL(12, 2);
    DECLARE v_cuenta_destino_existe INT DEFAULT 0;

    -- 2. Declaración del Manejador de Errores (Safety Net)
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SET p_resultado = 500;
        SET p_mensaje = 'Error crítico en el motor de base de datos. Transacción abortada.';
    END;

    -- 3. Validaciones defensivas de entrada
    IF p_monto <= 0.00 THEN
        SET p_resultado = 400;
        SET p_mensaje = 'El monto a transferir debe ser estrictamente positivo.';
        LEAVE sp_bloque;
    END IF;

    IF p_cuenta_origen = p_cuenta_destino THEN
        SET p_resultado = 400;
        SET p_mensaje = 'La cuenta de origen y destino no pueden ser idénticas.';
        LEAVE sp_bloque;
    END IF;

    -- 4. Inicio explícito de la transacción
    START TRANSACTION;

    -- 5. Bloqueo pesimista de la cuenta origen para evaluar saldo
    SELECT saldo INTO v_saldo_origen
    FROM cuentas
    WHERE id = p_cuenta_origen
    FOR UPDATE;

    IF v_saldo_origen IS NULL THEN
        ROLLBACK;
        SET p_resultado = 404;
        SET p_mensaje = 'La cuenta de origen no existe.';
        LEAVE sp_bloque;
    END IF;

    IF v_saldo_origen < p_monto THEN
        ROLLBACK;
        SET p_resultado = 422;
        SET p_mensaje = 'Saldo insuficiente para completar la operación.';
        LEAVE sp_bloque;
    END IF;

    -- 6. Verificar y bloquear cuenta destino
    SELECT COUNT(*) INTO v_cuenta_destino_existe
    FROM cuentas
    WHERE id = p_cuenta_destino
    FOR UPDATE;

    IF v_cuenta_destino_existe = 0 THEN
        ROLLBACK;
        SET p_resultado = 404;
        SET p_mensaje = 'La cuenta de destino no existe.';
        LEAVE sp_bloque;
    END IF;

    -- 7. Ejecutar los débitos y créditos
    UPDATE cuentas 
    SET saldo = saldo - p_monto 
    WHERE id = p_cuenta_origen;

    UPDATE cuentas 
    SET saldo = saldo + p_monto 
    WHERE id = p_cuenta_destino;

    -- 8. Auditoría obligatoria del movimiento
    INSERT INTO movimientos_bancarios (
        cuenta_origen_id, 
        cuenta_destino_id, 
        monto, 
        fecha_operacion
    ) VALUES (
        p_cuenta_origen, 
        p_cuenta_destino, 
        p_monto, 
        NOW()
    );

    -- 9. Confirmación permanente
    COMMIT;

    SET p_resultado = 200;
    SET p_mensaje = 'Transferencia completada exitosamente.';

END sp_bloque //

DELIMITER ;
```

---

## 6. Resumen de Directrices de Producción

1. **Toda operación multidominio exige transacción:** Si un flujo modifica más de una tabla o involucra validaciones críticas de saldo/inventario, debe envolverse en `START TRANSACTION ... COMMIT / ROLLBACK`.
2. **Siempre incluye un `EXIT HANDLER`:** Nunca dejes un procedimiento transaccional sin captura de excepciones; previene bloqueos residuales en el motor.
3. **Orden consistente en bloqueos para mitigar deadlocks:** Si dos transacciones necesitan acceder a las cuentas `A` y `B`, diseña la aplicación para que siempre bloqueen primero la cuenta con menor ID (`id_menor` antes que `id_mayor`).
4. **Mantén las transacciones ultracortas:** Adquiere bloqueos, actualiza los datos y haz `COMMIT` de inmediato. Jamás incluyas peticiones a APIs externas ni procesamiento intensivo de archivos dentro de una transacción activa de base de datos.
