# Comandos y Funciones en MySQL

![Engine](https://img.shields.io/badge/Engine-MySQL-4479A1?style=flat&logo=mysql&logoColor=white)

Guia de referencia rapida de comandos y funciones de uso frecuente en consola y consultas SQL.

---

## Funciones de Fecha y Hora

### `NOW()`

Devuelve la fecha y hora actual del sistema en el momento en que se ejecuta la sentencia.

- **Formato de retorno:** `'YYYY-MM-DD HH:MM:SS'` (o formato numerico `YYYYMMDDHHMMSS` si se usa en contexto numerico).

#### Sintaxis basica

```sql
SELECT NOW();
```

#### Utilidad y casos de uso

1. **Auditoria y marcas de tiempo (Timestamps):** Registrar el momento exacto en que se crea o actualiza un registro en la base de datos (campos como `created_at` o `updated_at`).

   ```sql
   INSERT INTO usuarios (nombre, email, created_at)
   VALUES ('Juan Perez', 'juan@example.com', NOW());
   ```

2. **Filtros y comparaciones temporales:** Consultar eventos futuros, pasados o calcular diferencias de tiempo respecto al momento actual.

   ```sql
   -- Obtener sesiones activas o no vencidas
   SELECT * FROM sesiones WHERE fecha_expiracion > NOW();
   ```

3. **Valores por defecto en DDL:** Configurar columnas para que tomen la fecha/hora actual automaticamente al insertar.

   ```sql
   CREATE TABLE pedidos (
       id INT AUTO_INCREMENT PRIMARY KEY,
       total DECIMAL(10, 2) NOT NULL,
       fecha_registro DATETIME DEFAULT NOW()
   );
   ```

---

## Variables de Sistema y Diagnostico

### `@@datadir`

Variable del sistema que almacena la ruta absoluta en disco donde el servidor de MySQL guarda fisicamente todas las bases de datos, tablas (`.ibd`), logs y el tablespace del sistema.

#### Sintaxis

```sql
SELECT @@datadir;
```

O tambien mediante la sentencia de variables:

```sql
SHOW VARIABLES LIKE 'datadir';
```

#### Utilidad y casos de uso

1. **Localizacion fisica de los datos:** Permite identificar en que particion o carpeta del sistema operativo se estan guardando los registros (crucial en Windows donde se ubica por defecto en la carpeta oculta `C:\ProgramData\MySQL\MySQL Server X.X\Data\`).
2. **Revision de almacenamiento y espacio en disco:** Imprescindible para DBAs y desarrolladores al auditar el tamano de los archivos `.ibd` de tablas pobladas o evaluar la migracion de almacenamiento a otro disco.
3. **Auditoria en migraciones y respaldos fisicos:** Facilita ubicar los directorios antes de realizar copias de seguridad directas o configurar respaldos en caliente (*cold/hot backups*).
