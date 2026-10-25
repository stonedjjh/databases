# 03. Claves Primarias, Foráneas e Integridad Referencial

![Engine](https://img.shields.io/badge/Engine-MySQL%208.0+-4479A1?style=flat&logo=mysql&logoColor=white)
![Focus](https://img.shields.io/badge/Focus-Primary%20Keys%20%26%20Referential%20Integrity-blue)

Las claves no son simplemente identificadores numéricos; en motores relacionales modernos como MySQL InnoDB, determinan **el ordenamiento físico de los datos en disco (Índice Agrupado / Clustered Index)** y garantizan que el sistema no contenga datos huérfanos o corrompidos.

---

## Índice

- [1. Claves Primarias (Primary Key - PK)](#1-claves-primarias-primary-key---pk)
  - [Claves Naturales vs Claves Sustitutas (Surrogate Keys)](#claves-naturales-vs-claves-sustitutas-surrogate-keys)
  - [El Debate de la Industria: `AUTO_INCREMENT` vs `UUIDv4` vs `UUIDv7`](#el-debate-de-la-industria-auto_increment-vs-uuidv4-vs-uuidv7)
  - [El Índice Agrupado (Clustered Index) en InnoDB y Fragmentación](#el-índice-agrupado-clustered-index-en-innodb-y-fragmentación)
- [2. Claves Foráneas (Foreign Key - FK) e Integridad Referencial](#2-claves-foráneas-foreign-key---fk-e-integridad-referencial)
  - [¿Qué significa la Integridad Referencial?](#qué-significa-la-integridad-referencial)
  - [Acciones de Eliminación y Actualización (`ON DELETE` / `ON UPDATE`)](#acciones-de-eliminación-y-actualización-on-delete--on-update)
  - [El Peligro Oculto de `ON DELETE CASCADE` en Producción](#el-peligro-oculto-de-on-delete-cascade-en-producción)
- [3. Claves Únicas (UNIQUE) vs Claves Primarias](#3-claves-únicas-unique-vs-claves-primarias)
- [4. Claves Compuestas (Composite Keys)](#4-claves-compuestas-composite-keys)
- [5. Resumen de Buenas Prácticas de Ingeniería](#5-resumen-de-buenas-prácticas-de-ingeniería)

---

## 1. Claves Primarias (Primary Key - PK)

Una **Clave Primaria** es una columna (o conjunto de columnas) que identifica de forma inequívoca cada registro dentro de una tabla. Debe cumplir dos propiedades matemáticas innegociables: **ser única** y **no admitir valores `NULL`**.

### Claves Naturales vs Claves Sustitutas (Surrogate Keys)

1. **Clave Natural:** Es un dato que ya existe en el mundo real y es único (ej. número de pasaporte, DNI/cédula, VIN de un vehículo, código ISBN de un libro).
   - *Riesgo en producción:* Los datos del mundo real cambian (un gobierno cambia el formato del pasaporte o una persona corrige un error de digitación). Modificar una clave natural propaga cambios destructivos por todas las tablas foráneas.
2. **Clave Sustituta (Recomendada):** Un identificador artificial generado por el sistema que no tiene significado intrínseco de negocio (ej. un entero secuencial `id` o un UUID).
   - *Ventaja:* Inmutable, ligero, no cambia jamás y desacopla la arquitectura técnica de las leyes del mundo real.

### El Debate de la Industria: `AUTO_INCREMENT` vs `UUIDv4` vs `UUIDv7`

| Criterio | `BIGINT AUTO_INCREMENT` | `UUIDv4` (Aleatorio puro) | `UUIDv7` (Temporalmente Ordenado) |
| :--- | :--- | :--- | :--- |
| **Tamaño en Almacenamiento** | **8 Bytes** (Ultracompacto) | **16 Bytes** (o 36 chars como texto) | **16 Bytes** (binario eficiente) |
| **Generación Descentralizada** | No (Requiere bloqueo central en la BD) | **Sí** (Cualquier microservicio o cliente lo genera) | **Sí** (Cualquier microservicio lo genera) |
| **Seguridad / Enumerable** | Inseguro (Expuesto en URLs permite raspar datos: `/user/1`, `/user/2`) | Seguro (Imposible de adivinar o enumerar) | Seguro (Imposible de enumerar) |
| **Impacto en B-Tree InnoDB** | Óptimo (Inserción al final del índice) | **Desastroso** (Fragmentación y *Page Splits*) | **Excelente** (Inserción secuencial ordenada por tiempo) |

> [!TIP]
> **Estándar Moderno:** Si usas UUIDs en MySQL 8.0+, almacénalos en columnas `BINARY(16)` y utiliza UUIDs secuenciales (como **UUIDv7**) para evitar que las inserciones destruyan el caché de memoria (`Buffer Pool`).

### El Índice Agrupado (Clustered Index) en InnoDB

En MySQL InnoDB, la tabla **ES** físicamente el árbol B-Tree de la Clave Primaria. 
- Cada vez que insertas una fila con un entero auto-incremental, la fila se escribe ordenadamente en la última página de disco.
- Si insertas un `UUIDv4` aleatorio, InnoDB tiene que saltar a páginas intermedias en memoria/disco, dividir la página en dos (*Page Split*), mover datos y fragmentar el índice, reduciendo la velocidad de escritura hasta en un 80% en tablas grandes.

---

## 2. Claves Foráneas (Foreign Key - FK) e Integridad Referencial

Una **Clave Foránea** es una columna en una tabla que apunta y referencia directamente a la Clave Primaria de otra tabla.

### ¿Qué significa la Integridad Referencial?

Garantiza que el motor rechace cualquier operación que intente crear una relación rota:
- **No puedes insertar un pedido** con un `cliente_id = 999` si el cliente 999 no existe.
- **No puedes eliminar un cliente** si tiene pedidos registrados en el sistema, a menos que se defina explícitamente cómo debe resolverse el borrado.

### Acciones de Eliminación y Actualización (`ON DELETE` / `ON UPDATE`)

Al definir una `FOREIGN KEY`, debes especificar qué comportamiento debe adoptar MySQL cuando el registro padre sea alterado o borrado:

| Acción | Comportamiento | Cuándo Usarlo |
| :--- | :--- | :--- |
| **`RESTRICT` / `NO ACTION`** *(Por defecto)* | **Bloquea y aborta** la eliminación o actualización del registro padre arrojando un error si existen hijos vinculados. | **Uso bancario y contable:** Jamás debes poder borrar un cliente si tiene facturas o pagos asociados. |
| **`CASCADE`** | Al borrar o actualizar el padre, **borra o actualiza automáticamente** todas las filas hijas vinculadas. | Tablas dependientes de existencia (ej. borrar una `factura` borra sus `detalle_factura`). |
| **`SET NULL`** | Establece el valor de la columna foránea en `NULL` en los registros hijos (requiere que la columna permita `NULL`). | Cuando el hijo puede sobrevivir sin el padre (ej. si se borra un `vendedor`, el `pedido` queda con `vendedor_id = NULL`). |

### El Peligro Oculto de `ON DELETE CASCADE` en Producción

```sql
-- ¡CUIDADO! Un DELETE inocente puede vaciar la base de datos completa:
DELETE FROM organizaciones WHERE id = 5;
```
Si la tabla `organizaciones` tiene `CASCADE` hacia `usuarios`, y `usuarios` tiene `CASCADE` hacia `pedidos`, y `pedidos` tiene `CASCADE` hacia `pagos`, ejecutar ese único comando borrará millones de transacciones históricas sin pedir confirmación.

> [!CAUTION]
> En entornos de producción reales, se prefiere **`ON DELETE RESTRICT`** para entidades críticas de negocio y se aplican técnicas de **Borrado Lógico (*Soft Delete*)** mediante columnas como `deleted_at TIMESTAMP NULL`.

---

## 3. Claves Únicas (UNIQUE) vs Claves Primarias

| Característica | Clave Primaria (`PRIMARY KEY`) | Clave Única (`UNIQUE`) |
| :--- | :---: | :---: |
| **Cantidad por tabla** | Exactamente **1** | Múltiples permitidas |
| **Valores `NULL`** | Estrictamente prohibidos (`NOT NULL`) | Permite múltiples `NULL` en el estándar SQL |
| **Organización Física** | Define el *Clustered Index* en InnoDB | Crea un índice secundario (*Secondary B-Tree*) |
| **Uso común** | Identificador maestro interno del sistema | Restricciones de negocio (`email`, `cuit_rut`, `slug`) |

---

## 4. Claves Compuestas (Composite Keys)

Una clave compuesta utiliza dos o más columnas simultáneamente para garantizar la unicidad.

```sql
CREATE TABLE roles_permisos (
    rol_id INT NOT NULL,
    permiso_id INT NOT NULL,
    PRIMARY KEY (rol_id, permiso_id), -- Clave Compuesta
    CONSTRAINT fk_rp_rol FOREIGN KEY (rol_id) REFERENCES roles(id) ON DELETE CASCADE,
    CONSTRAINT fk_rp_permiso FOREIGN KEY (permiso_id) REFERENCES permisos(id) ON DELETE CASCADE
) ENGINE=InnoDB;
```

---

## 5. Resumen de Buenas Prácticas de Ingeniería

1. **Usa `BIGINT AUTO_INCREMENT` o `UUIDv7` como Clave Primaria:** Evita `UUIDv4` como Clave Primaria agrupada.
2. **Nombra tus claves foráneas explícitamente:** Usa la convención `fk_<tabla_origen>_<tabla_destino>` para facilitar la depuración de errores de violación de constraints.
3. **Indexa SIEMPRE las Claves Foráneas:** Aunque MySQL crea un índice automáticamente al definir un `FOREIGN KEY`, asegúrate de que exista siempre un índice B-Tree en esa columna para optimizar los `JOINs`.
4. **Sé conservador con `CASCADE`:** Usa `RESTRICT` por defecto para proteger la integridad contable y legal de tu base de datos.
