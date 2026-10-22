# Comandos y Funciones en MySQL

![Engine](https://img.shields.io/badge/Engine-MySQL%208.0+-4479A1?style=flat&logo=mysql&logoColor=white)

Guía de referencia rápida de comandos, funciones de uso frecuente y variables del sistema para diagnóstico y administración.

---

## Índice de Contenido

- [1. Funciones de Fecha y Hora](#1-funciones-de-fecha-y-hora)
  - [`NOW()`](#now)
- [2. Variables de Sistema y Diagnóstico (`@@`)](#2-variables-de-sistema-y-diagnóstico-)
  - [El Prefijo `@@`: Variables de Sistema vs Variables de Usuario](#el-prefijo--variables-de-sistema-vs-variables-de-usuario)
  - [`@@datadir`](#datadir)
  - [`@@lower_case_table_names`](#lower_case_table_names)
- [3. Diagnóstico Rápido de Sesión y Servidor](#3-diagnóstico-rápido-de-sesión-y-servidor)

---

## 1. Funciones de Fecha y Hora

### `NOW()`

Devuelve la fecha y hora actual del sistema en el momento en que se ejecuta la sentencia.

- **Formato de retorno:** `'YYYY-MM-DD HH:MM:SS'` (o formato numérico `YYYYMMDDHHMMSS` si se usa en contexto numérico).

#### Sintaxis básica

```sql
SELECT NOW();
```

#### Utilidad y casos de uso

1. **Auditoría y marcas de tiempo (Timestamps):** Registrar el momento exacto en que se crea o actualiza un registro en la base de datos (campos como `created_at` o `updated_at`).

   ```sql
   INSERT INTO usuarios (nombre, email, created_at)
   VALUES ('Juan Perez', 'juan@example.com', NOW());
   ```

2. **Filtros y comparaciones temporales:** Consultar eventos futuros, pasados o calcular diferencias de tiempo respecto al momento actual.

   ```sql
   -- Obtener sesiones activas o no vencidas
   SELECT * FROM sesiones WHERE fecha_expiracion > NOW();
   ```

3. **Valores por defecto en DDL:** Configurar columnas para que tomen la fecha/hora actual automáticamente al insertar.

   ```sql
   CREATE TABLE pedidos (
       id INT AUTO_INCREMENT PRIMARY KEY,
       total DECIMAL(10, 2) NOT NULL,
       fecha_registro DATETIME DEFAULT NOW()
   );
   ```

---

## 2. Variables de Sistema y Diagnóstico (`@@`)

### El Prefijo `@@`: Variables de Sistema vs Variables de Usuario

En MySQL, el uso de `@` y `@@` tiene significados completamente distintos:
- **`@variable` (un solo arroba):** Variables definidas por el **usuario** en la sesión actual (`SET @mi_id = 10;`). Son temporales y se destruyen al cerrar la conexión.
- **`@@variable` (doble arroba):** Variables de **sistema o configuración interna del motor** (pueden ser de ámbito global `@@GLOBAL.nombre` o de sesión `@@SESSION.nombre`).

Para consultar cualquier parámetro del motor de forma inmediata en una consulta SQL:

```sql
SELECT @@nombre_variable;
-- Ejemplo:
SELECT @@version, @@port, @@autocommit;
```

---

### `@@datadir`

Variable del sistema que almacena la ruta absoluta en disco donde el servidor de MySQL guarda físicamente todas las bases de datos, tablas (`.ibd`), logs y el tablespace del sistema.

#### Sintaxis

```sql
SELECT @@datadir;
```

O también mediante la sentencia general:

```sql
SHOW VARIABLES LIKE 'datadir';
```

#### Utilidad y casos de uso

1. **Localización física de los datos:** Permite identificar en qué partición o carpeta del sistema operativo se están guardando los registros (crucial en Windows donde se ubica por defecto en la carpeta oculta `C:\ProgramData\MySQL\MySQL Server X.X\Data\`).
2. **Revisión de almacenamiento y espacio en disco:** Imprescindible para DBAs y desarrolladores al auditar el tamaño de los archivos `.ibd` de tablas pobladas o evaluar la migración de almacenamiento a otro disco.
3. **Auditoría en migraciones y respaldos físicos:** Facilita ubicar los directorios antes de realizar copias de seguridad directas o configurar respaldos en caliente (*cold/hot backups*).

---

### `@@lower_case_table_names`

Variable fundamental del motor que define cómo MySQL gestiona la **sensibilidad a mayúsculas y minúsculas (case sensitivity)** en los nombres de esquemas (bases de datos) y tablas en el sistema de archivos del sistema operativo.

#### Sintaxis

```sql
SELECT @@lower_case_table_names;
-- O explícitamente a nivel global:
SELECT @@GLOBAL.lower_case_table_names;
```

#### Valores posibles y comportamiento

| Valor | Comportamiento | Sistema Operativo Típico |
| :---: | :--- | :--- |
| **`0`** | **Case-sensitive (Sensible a mayúsculas):** Los nombres de tablas y bases de datos se almacenan en disco exactamente como se escriben. `Usuarios` y `usuarios` son dos tablas distintas. | **Linux por defecto** |
| **`1`** | **Case-insensitive (Insensible / Forzado a minúsculas):** Los nombres de tablas se almacenan siempre en **minúsculas** en disco y las consultas se comparan en minúsculas sin importar cómo se escriban. | **Windows por defecto** |
| **`2`** | Los nombres se almacenan en disco con las mayúsculas/minúsculas originales declaradas, pero las comparaciones en consultas SQL se resuelven en minúsculas. | **macOS** (HFS+/APFS no sensibles a mayúsculas) |

> [!NOTE]
> Los nombres de **columnas**, índices y alias nunca son sensibles a mayúsculas/minúsculas en ningún sistema operativo ni configuración.

#### ⚠️ Advertencia Crítica en MySQL 8.0+
- **Inmutable en caliente:** Esta variable **no se puede modificar** con `SET GLOBAL` ni cambiando simplemente el archivo `my.ini` / `my.cnf` una vez arrancado el servidor.
- **Definición única al inicializar:** Solo puede ser establecida durante la inicialización del directorio de datos del servidor (`mysqld --initialize`).
- Si se intenta forzar un cambio de valor en `my.ini` después de que el catálogo de datos ya fue creado, el servicio de MySQL **fallará al iniciar** por discrepancia de diccionario.

#### ¿Por qué es crucial para el programador?
Evita el típico error en despliegues: un código desarrollado en Windows donde escribir `SELECT * FROM Usuarios;` funciona sin problemas (gracias a `@@lower_case_table_names = 1`), pero que al desplegar en un servidor de producción Linux (`@@lower_case_table_names = 0`) arroja:
```text
ERROR 1146 (42S02): Table 'tienda.Usuarios' doesn't exist (la tabla física en disco se llama 'usuarios')
```

---

## 3. Diagnóstico Rápido de Sesión y Servidor

Consulta combinada recomendada para auditar rápidamente el entorno de trabajo actual:

```sql
SELECT 
    @@version AS version_mysql,
    @@datadir AS directorio_datos,
    @@lower_case_table_names AS sensibilidad_mayusculas,
    @@autocommit AS autocommit_activo,
    DATABASE() AS base_datos_actual,
    CURRENT_USER() AS usuario_conectado;
```
