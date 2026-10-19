# Anatomia del Almacenamiento en MySQL (InnoDB)

![Engine](https://img.shields.io/badge/Engine-MySQL-4479A1?style=flat&logo=mysql&logoColor=white)
![Storage](https://img.shields.io/badge/Storage-InnoDB%20Internals-red)

Guia tecnica sobre como el motor de almacenamiento traduce las sentencias SQL en estructuras fisicas (carpetas y archivos binarios) dentro del disco duro.

---

## Indice de Contenido

- [1. La Base de Datos en Disco (Directorios)](#1-la-base-de-datos-en-disco-directorios)
- [2. La Tabla en Disco (Archivos .ibd)](#2-la-tabla-en-disco-archivos-ibd)
- [3. Estructura Interna de un Archivo .ibd](#3-estructura-interna-de-un-archivo-ibd)
- [4. Impacto Practico para el Programador: DROP vs DELETE vs TRUNCATE](#4-impacto-practico-para-el-programador-drop-vs-delete-vs-truncate)
- [5. El Ciclo de Vida de un DELETE: Purge Thread y Recuperacion](#5-el-ciclo-de-vida-de-un-delete-purge-thread-y-recuperacion)

---

## 1. La Base de Datos en Disco (Directorios)

Cuando en SQL se ejecuta la creacion de un esquema o base de datos:

```sql
CREATE DATABASE tienda_online;
```

A nivel de sistema operativo ocurre una operacion directa sobre el sistema de archivos: el motor crea un nuevo directorio con ese mismo nombre dentro de la ruta definida por `@@datadir`:

```text
C:\ProgramData\MySQL\MySQL Server 8.0\Data\tienda_online\
```

Una base de datos en MySQL es, fisicamente, una carpeta que agrupa los archivos correspondientes a sus tablas y objetos.

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

Este archivo recibe el nombre de **Tablespace individual** (habilitado por defecto mediante la directiva `innodb_file_per_table = ON`). Cada tabla almacena de forma aislada tanto sus registros como sus indices.

---

## 3. Estructura Interna de un Archivo .ibd

Un archivo `.ibd` no es texto plano; es un archivo binario de acceso aleatorio de alto rendimiento organizado en bloques fijos:

1. **Paginas de 16 KB:** Es la unidad minima de lectura y escritura de InnoDB. Toda operacion de lectura desde disco a memoria RAM (el *InnoDB Buffer Pool*) se realiza en bloques de 16 KB.
2. **Indice Agrupado (Clustered Index / B+ Tree):** En InnoDB, los datos de la tabla no estan dispersos: estan fisicamente ordenados dentro de la estructura de arbol B+ del indice de la clave primaria (`PRIMARY KEY`). La clave primaria es la propia tabla.
3. **Indices Secundarios:** Si se crean indices sobre otros campos (por ejemplo, sobre `email`), estos indices secundarios se almacenan dentro del mismo archivo `.ibd`, apuntando al valor de la clave primaria correspondiente.

---

## 4. Impacto Practico para el Programador: DROP vs DELETE vs TRUNCATE

Entender que las tablas son archivos fisicos resuelve uno de los problemas mas comunes en entornos de desarrollo y produccion: **la recuperacion de espacio en disco**.

| Sentencia | Operacion Fisica en Disco | Recupera Espacio Inmediato en SO? |
| :--- | :--- | :--- |
| `DROP TABLE usuarios;` | Elimina por completo el archivo `usuarios.ibd` del sistema de archivos. | **Si**, de forma instantanea. |
| `TRUNCATE TABLE usuarios;` | Destruye el archivo `.ibd` original y crea un nuevo archivo `.ibd` vacio con el tamano base inicial (~112 KB). | **Si**, de forma instantanea. |
| `DELETE FROM usuarios;` | Marca las filas o paginas internas como "reutilizables" dentro del arbol, pero **no devuelve el espacio al sistema operativo**. El archivo `.ibd` conserva su tamano maximo en MB/GB. | **No**, el archivo en disco no se reduce. |

> [!WARNING]
> Si una tabla alcanza 50 GB y se ejecuta un `DELETE` masivo sin clausula `WHERE`, las filas desapareceran de las consultas, pero en el disco duro el archivo `.ibd` seguira ocupando 50 GB. Para forzar la devolucion del espacio libre al sistema operativo es necesario reconstruir la tabla mediante `OPTIMIZE TABLE nombre_tabla;` o `ALTER TABLE nombre_tabla ENGINE=InnoDB;`.

---

## 5. El Ciclo de Vida de un DELETE: Purge Thread y Recuperacion

Cuando se ejecuta un `DELETE`, el motor no destruye los datos de inmediato en milisegundos:

1. **Delete-Marking (Marcado Logico):** El registro recibe una bandera (*delete-mark*). La fila sigue fisicamente en la pagina porque transacciones concurrentes con aislamiento MVCC podrian necesitar seguir leyendola.
2. **Purge Thread (Garbage Collector de InnoDB):** Un hilo en segundo plano verifica periodicamente si quedan transacciones activas que requieran la version anterior. Si ya nadie la necesita, el Purge Thread limpia los punteros y anade el espacio a una lista interna de bloques libres (*Free List*).
3. **Reutilizacion Interna:** El espacio liberado queda reservado exclusivamente para futuras inserciones (`INSERT`) de esa misma tabla, sin devolverle los bytes al sistema operativo.

### Es posible recuperar datos tras un DELETE accidental?

- **Con Transacciones Activas:** Si la sesion trabajo con `START TRANSACTION;` y aun no se ha hecho `COMMIT;`, un simple `ROLLBACK;` restaura todo usando los *Undo Logs*.
- **Despues del COMMIT:**
  - A nivel de SQL estandar no es reversible de forma directa.
  - **Forense / Bajo Nivel:** Mientras el *Purge Thread* no haya purgado las paginas ni se hayan sobreescrito con nuevos registros, los bytes siguen existiendo fisicamente en el archivo `.ibd` (existen herramientas forenses de recuperacion de bloques).
  - **Estrategia Profesional (Binary Log):** En servidores con `binlog_format=ROW`, cada fila eliminada queda guardada como evento en el registro binario. Mediante la herramienta de consola `mysqlbinlog` es posible realizar un "rollback inverso" (transformar los `DELETE` en `INSERT`) para reconstruir la informacion. *(Nota: La configuracion y auditoria del Binlog se profundizara en los modulos avanzados de administracion).*
