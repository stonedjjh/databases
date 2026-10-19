# Conceptos Globales de Bases de Datos

![Topic](https://img.shields.io/badge/Topic-Database%20Fundamentals-informational)

Documento base de definiciones. Se ira completando progresivamente segun avance el estudio.

---

## 1. Que es una Base de Datos (DB)

Una base de datos es un conjunto organizado y estructurado de datos almacenados de forma digital en un sistema informatico, disenado para permitir su facil almacenamiento, consulta, actualizacion y administracion.

---

## 2. Que es un DBMS (Database Management System)

Un DBMS (Sistema de Gestion de Bases de Datos) es el software que sirve de interfaz entre la base de datos, los usuarios y los programas que necesitan acceder a ella. Permite crear, mantener, gestionar y controlar el acceso a los datos de manera eficiente y segura.

---

## 3. Principios ACID en Bases de Datos Relacionales

El modelo relacional se fundamenta en el concepto de **transaccion**: una unidad logica de trabajo compuesta por una o varias operaciones SQL que deben ejecutarse bajo las garantias **ACID** para asegurar la integridad de los datos ante fallos, errores o concurrencia masiva.

### Las 4 Propiedades Fundamentales

1. **Atomicidad (Atomicity) - *"Todo o Nada"*:**
   - Una transaccion no puede quedar a medias. O se confirman todas sus operaciones (`COMMIT`) o el motor revierte absolutamente todos los cambios (`ROLLBACK`) ante cualquier error.
   - *Ejemplo real:* En una transferencia bancaria, se descuenta dinero de la cuenta origen y se acredita en la cuenta destino. Si el sistema falla a mitad del proceso, la atomicidad garantiza que el dinero no desaparezca en el limbo; el saldo vuelve a su estado inicial.

2. **Consistencia (Consistency) - *"Respeto a las Reglas"*:**
   - Toda transaccion debe llevar a la base de datos de un estado valido a otro estado valido, respetando todas las restricciones de integridad definidas en el esquema (claves primarias unicas, claves foraneas existentes, tipos de datos y restricciones `CHECK`).
   - *Ejemplo real:* Si una regla dicta que el saldo de una cuenta nunca puede ser menor a cero, cualquier transaccion que intente dejar el saldo en negativo es abortada y rechazada por el motor.

3. **Aislamiento (Isolation) - *"Sin Interferencias Concurrentes"*:**
   - Las operaciones simultaneas de multiples usuarios no deben interferir entre si. El resultado de ejecutar transacciones en paralelo debe ser identico al de ejecutarlas de forma estrictamente secuencial.
   - **Prevencion de Condiciones de Carrera (Race Conditions):** Evita escenarios criticos de concurrencia donde dos procesos leen y modifican el mismo recurso al mismo tiempo (por ejemplo, dos usuarios intentando comprar simultaneamente el ultimo producto en stock). Mediante bloqueos (*locks*) y niveles de aislamiento (como MVCC), el motor garantiza que una transaccion espere o se aisle adecuadamente de la otra.

   > [!TIP]
   > **Pro-Tip de Arquitectura (Bloqueo Optimista / Optimistic Locking):**
   > Para evitar bloqueos pesados en la base de datos (`SELECT ... FOR UPDATE`), un patron profesional de concurrencia consiste en utilizar un campo de control como `updated_at` (o un numero de `version`).
   > 
   > Al emitir la actualizacion, el backend incluye el valor leido previamente en la clausula de filtro:
   > ```sql
   > UPDATE productos 
   > SET stock = stock - 1, updated_at = NOW()
   > WHERE id = 10 AND updated_at = '2026-10-18 20:00:00';
   > ```
   > Si otro usuario modifico el registro un milisegundo antes, el `updated_at` en disco ya no coincidira y la consulta afectara a **0 filas** (`affected_rows = 0`). La aplicacion detecta esto inmediatamente, aborta la operacion y solicita al usuario refrescar los datos, eliminando la condicion de carrera sin degradar el rendimiento del motor.

4. **Durabilidad (Durability) - *"Persistencia Garantizada"*:**
   - Una vez que la base de datos confirma el exito de una transaccion (`COMMIT`), los cambios son permanentes e indestructibles por caidas del software o cortes de energia electrica en el servidor.
   - *Como se logra:* Los cambios se escriben inmediatamente en registros de recuperacion en disco antes de confirmarle al cliente (como el *Redo Log* en MySQL InnoDB o el *WAL - Write-Ahead Logging* en PostgreSQL). Si el servidor se apaga repentinamente, al reiniciar reconstruye el estado exacto a partir de ese registro.
