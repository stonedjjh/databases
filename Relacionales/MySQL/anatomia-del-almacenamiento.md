# Anatomía del Almacenamiento en MySQL (InnoDB)

![Engine](https://img.shields.io/badge/Engine-MySQL%208.0+-4479A1?style=flat&logo=mysql&logoColor=white)
![Storage](https://img.shields.io/badge/Storage-InnoDB%20Internals-red)
![Verification](https://img.shields.io/badge/Verified-MySQL%208.0-success)

Guía técnica sobre cómo el motor de almacenamiento traduce las sentencias SQL en estructuras físicas (directorios y archivos binarios) dentro del disco duro, su impacto en operaciones de borrado y recuperación de espacio.

---

## Índice de Contenido

- [1. La Base de Datos en Disco (Directorios)](#1-la-base-de-datos-en-disco-directorios)
- [2. La Tabla en Disco (Archivos .ibd)](#2-la-tabla-en-disco-archivos-ibd)
- [3. Estructura Interna de un Archivo .ibd](#3-estructura-interna-de-un-archivo-ibd)
- [4. Impacto Práctico para el Programador: DROP vs DELETE vs TRUNCATE](#4-impacto-práctico-para-el-programador-drop-vs-delete-vs-truncate)
- [5. El Ciclo de Vida de un DELETE: Purge Thread y Reutilización de Espacio](#5-el-ciclo-de-vida-de-un-delete-purge-thread-y-reutilización-de-espacio)
- [6. ¿Es Posible Recuperar Datos tras un DELETE Accidental?](#6-es-posible-recuperar-datos-tras-un-delete-accidental)
- [7. Cuadro Comparativo: ¿Cómo Funciona en PostgreSQL?](#7-cuadro-comparativo-cómo-funciona-en-postgresql)

---

## 1. La Base de Datos en Disco (Directorios)

Cuando en SQL se ejecuta la creación de un esquema o base de datos:

```sql
CREATE DATABASE tienda_online;
```

A nivel de sistema operativo ocurre una operación directa sobre el sistema de archivos: el motor crea un nuevo directorio con ese mismo nombre dentro de la ruta definida por `@@datadir`:

```text
C:\ProgramData\MySQL\MySQL Server 8.0\Data\tienda_online\
```

> [!NOTE]
> **Windows vs Linux y `lower_case_table_names`:**  
> La sensibilidad a mayúsculas y minúsculas en nombres de bases de datos y tablas depende de la variable de sistema `lower_case_table_names` configurada al inicializar el cluster de MySQL:
> - `0` (Linux por defecto): Los nombres se almacenan tal cual y son sensibles a mayúsculas/minúsculas.
> - `1` (Windows por defecto): Los nombres se almacenan y comparan en minúsculas.
> - `2` (macOS): Se almacenan como se declaran pero se comparan en minúsculas.  
> Los nombres de **columnas** nunca distinguen mayúsculas/minúsculas en ningún sistema operativo.

---

## 2. La Tabla en Disco (Archivos .ibd)

Cuando se crea una tabla utilizando el motor predeterminado InnoDB:

```sql
CREATE TABLE usuarios (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(100),
    email VARCHAR(150)
);
```

MySQL genera en disco un archivo binario dentro de la carpeta de la base de datos:

```text
.../tienda_online/usuarios.ibd
```

Este archivo recibe el nombre de **Tablespace individual** (habilitado por defecto mediante la directiva `innodb_file_per_table = ON`). Cada tabla almacena de forma aislada tanto sus registros como sus índices.

---

## 3. Estructura Interna de un Archivo .ibd

Un archivo `.ibd` no es texto plano; es un archivo binario de acceso aleatorio de alto rendimiento organizado en bloques fijos:

1. **Páginas de 16 KB:** Es la unidad mínima de lectura y escritura de InnoDB. Toda operación de lectura desde disco a memoria RAM (el *InnoDB Buffer Pool*) se realiza en páginas de 16 KB.
2. **Índice Agrupado (Clustered Index / B+ Tree):**  
   - En InnoDB, la clave primaria (`PRIMARY KEY`) **es** la propia tabla.
   - Las páginas hoja (*leaf pages*) están enlazadas secuencialmente en orden de la clave primaria.
   - *Detalle interno:* Dentro de cada página de 16 KB, los registros no están físicamente contiguos en disco, sino ordenados mediante una lista enlazada lógica de punteros de registro (*heap directory slots*).
   - *¿Y si no defines PRIMARY KEY?* InnoDB busca el primer índice `UNIQUE` cuyas columnas sean `NOT NULL`. Si no existe ninguno, InnoDB genera un identificador oculto del sistema de 6 bytes (`GEN_CLUST_INDEX` o *Row ID* secuencial).
3. **Índices Secundarios:** Si se crean índices sobre otros campos (por ejemplo, sobre `email`), estos índices secundarios se almacenan dentro del mismo archivo `.ibd`. Cada nodo hoja del índice secundario no guarda la fila completa, sino que apunta al valor de la `PRIMARY KEY` correspondiente.

---

## 4. Impacto Práctico para el Programador: DROP vs DELETE vs TRUNCATE

Entender que las tablas son archivos físicos resuelve uno de los problemas más comunes en entornos de desarrollo y producción: **la persistencia, reversibilidad y recuperación de espacio en disco**.

| Sentencia | Tipo de Sentencia | Operación Física en Disco | ¿Recupera Espacio Inmediato en SO? | ¿Se puede deshacer con `ROLLBACK`? |
| :--- | :--- | :--- | :--- | :--- |
| `DROP TABLE usuarios;` | **DDL** | Elimina por completo el archivo `usuarios.ibd` del sistema de archivos. | **Sí**, de forma casi instantánea. | **No**. Causa un **COMMIT implícito**. |
| `TRUNCATE TABLE usuarios;` | **DDL** | Destruye el archivo `.ibd` original y crea uno nuevo vacío (~112 KB base). Reinicia el contador `AUTO_INCREMENT` a 1. | **Sí**, de forma casi instantánea. | **No**. Causa un **COMMIT implícito**. *(Falla si otra tabla la referencia con una FK activa).* |
| `DELETE FROM usuarios;` | **DML** | Marca las filas como "borradas" en el árbol B+ y genera registros de *Undo Log* y *Binlog* fila por fila. | **No**, el archivo `.ibd` conserva su tamaño máximo en MB/GB. | **Sí**, siempre que `autocommit = OFF` o se ejecute dentro de un `START TRANSACTION;` explícito. |

> [!CAUTION]
> ### Riesgo de Colapso con `OPTIMIZE TABLE` en Tablas Grandes
> Si una tabla alcanza **50 GB** y ejecutas un `DELETE` masivo, el archivo `.ibd` en disco seguirá ocupando 50 GB.  
> Para devolver ese espacio libre al sistema operativo es necesario reconstruir la tabla mediante `OPTIMIZE TABLE usuarios;` (o `ALTER TABLE usuarios ENGINE=InnoDB;`).  
> **ADVERTENCIA CRÍTICA:** `OPTIMIZE TABLE` realiza una reconstrucción creando un nuevo archivo temporal en disco y copiando los registros activos. Si tu disco no cuenta con al menos **50 GB de espacio libre adicional**, el comando **saturará el disco al 100% y colapsará el servidor de base de datos**.

---

## 5. El Ciclo de Vida de un DELETE: Purge Thread y Reutilización de Espacio

Cuando se ejecuta un `DELETE`, el motor no destruye los datos de inmediato:

1. **Delete-Marking (Marcado Lógico):** El registro recibe un flag de borrado (*delete-mark*). La fila sigue físicamente en la página porque transacciones concurrentes bajo aislamiento MVCC (*Multi-Version Concurrency Control*) podrían necesitar seguir leyéndola.
2. **Purge Thread (Garbage Collector de InnoDB):** Un hilo en segundo plano verifica periódicamente si existen transacciones activas más antiguas que requieran la versión anterior. Si ninguna transacción la necesita, el Purge Thread limpia los punteros y añade el espacio a una lista interna de bloques libres (*Free List*).
3. **Matiz sobre la Reutilización Interna:**  
   - El espacio liberado queda reservado **únicamente para la misma tabla**.
   - **Limitación con claves autoincrementales:** Si tu tabla usa una `PRIMARY KEY AUTO_INCREMENT`, las nuevas inserciones siempre se agregan secuencialmente al final del árbol B+ (en la página de mayor valor). El espacio libre dejado por un `DELETE` en medio de la tabla **solo se reutilizará si se insertan registros cuyos IDs caigan exactamente en ese rango** o en sus índices secundarios.

---

## 6. ¿Es Posible Recuperar Datos tras un DELETE Accidental?

### Escenario A: Con Transacción Abierta
Si la sesión inició con `START TRANSACTION;` y aún no se ha hecho `COMMIT;`, un simple `ROLLBACK;` restaura todo de forma instantánea gracias a los *Undo Logs*.

### Escenario B: Con `autocommit = ON` (Comportamiento Predeterminado)
En MySQL y clientes SQL por defecto, `autocommit` está en `1` (`ON`). Cada instrucción individual ejecuta un `COMMIT` automático e implícito al terminar. Si lanzas un `DELETE` sin `START TRANSACTION;`, el borrado queda confirmado inmediatamente y `ROLLBACK;` no tendrá ningún efecto.

### Escenario C: Recuperación Profesional tras el COMMIT
Una vez confirmado el `COMMIT`:
1. **Forense / Nivel de Bytes:** Mientras el *Purge Thread* no haya purgado las páginas ni se hayan sobreescrito con nuevas escrituras, los bytes siguen existiendo físicamente en el archivo `.ibd`.
2. **Registro Binario (Binary Log):**
   - En **MySQL 8.0+**, el Binlog viene **activado por defecto** (`log_bin = ON`) con formato basado en filas (`binlog_format = ROW`).
   - El comando nativo `mysqlbinlog -v` extrae los eventos en pseudo-SQL con comentarios, pero **no genera sentencias `INSERT` directamente listas para ejecutar**.
   - Para generar un "rollback inverso" automatizado (convertir eventos `DELETE` en `INSERT`), se requieren utilidades de análisis forense de binlog especializadas (como `binlog2sql` o `my2sql`).
3. **Estrategia Estándar de la Industria (PITR - Point-In-Time Recovery):**  
   La verdadera recuperación profesional en producción no depende de trucos forenses:
   - Se toma el último backup consistente (ej. con `mysqldump` o Percona XtraBackup).
   - Se restaura en un servidor o base de datos temporal de rescate.
   - Se reproduce el Binlog con `mysqlbinlog --stop-datetime` o `--stop-position` hasta **un microsegundo antes** de la sentencia `DELETE` errónea.

---

## 7. Cuadro Comparativo: ¿Cómo Funciona en PostgreSQL?

| Característica | MySQL (InnoDB) | PostgreSQL |
| :--- | :--- | :--- |
| **Estructura física de tabla** | **Clustered Index:** La tabla está físicamente ordenada en el árbol B+ de la PK (`.ibd`). | **Heap Tables:** Los datos se guardan sin orden físico específico; los índices son archivos separados. |
| **Comportamiento de DELETE** | Delete-marking + Purge Thread en segundo plano. | Genera **tuplas muertas** (*dead tuples*). Requiere `VACUUM` para marcarlas reutilizables. |
| **Devolver espacio al Sistema Operativo** | `OPTIMIZE TABLE` (reconstruye tabla y requiere espacio libre). | `VACUUM FULL` (bloquea la tabla y reescribe el archivo físico) o herramientas como `pg_repack`. |
| **¿`TRUNCATE` soporta ROLLBACK?** | **No**. Es DDL con COMMIT implícito en MySQL. | **Sí**. En PostgreSQL el DDL es transaccional; puedes hacer `ROLLBACK` de un `TRUNCATE` dentro de un `BEGIN;`. |
