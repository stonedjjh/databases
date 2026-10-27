# Bases de Datos Relacionales

![Status](https://img.shields.io/badge/Status-Completado-brightgreen)
![Score](https://img.shields.io/badge/Evaluaci%C3%B3n%20SQL-168%20%7C%20Avanzado-blueviolet)
![Engine](https://img.shields.io/badge/Engine-MySQL%208.0+-4479A1?style=flat&logo=mysql&logoColor=white)
![Certificate](https://img.shields.io/badge/Certificado-Udemy-blue?logo=udemy)

Modulo especializado en arquitectura, administracion, modelado formal e ingenieria de bases de datos relacionales con MySQL 8.0+.

---

## Logros y Certificaciones

- **Certificado del Curso:** [SQL Total - Domina Bases de Datos de 0 a Avanzado en 12 Dias (Udemy)](https://www.udemy.com/certificate/UC-3705a432-c742-43f1-832c-0c7b30ade390/)
- **Copia del Certificado:** [SQL Total.jpg](docs/SQL%20Total.jpg)
- **Evaluacion de Habilidades (SQL):** **168 / 200 (Nivel Avanzado - Top 22%)** ([Comprobante](docs/evaluacion.png))

---

## Estructura de Guias Tecnicas

- [00-conceptos-globales.md](00-conceptos-globales.md): Fundamentos de Bases de Datos, motores y Sistemas Gestores (DBMS).
- **MySQL/**:
  - [anatomia-del-almacenamiento.md](MySQL/anatomia-del-almacenamiento.md): Mapeo a disco (`.ibd`), tablespaces y estructura de paginas InnoDB.
  - [comandos.md](MySQL/comandos.md): Funciones del sistema (`NOW()`, `CURRENT_USER()`), diagnostico y variables globales (`@@datadir`, `@@lower_case_table_names`).
  - [lenguaje-sql-ddl-dml-dcl-tcl.md](MySQL/lenguaje-sql-ddl-dml-dcl-tcl.md): Taxonomia del lenguaje SQL, auto-commit en DDL, operaciones CRUD y control transaccional/seguridad.
  - [consultas.md](MySQL/consultas.md): Proyecciones, filtros condicionales, funciones nativas, `GROUP BY`, `HAVING` y subconsultas.
  - [joins-y-uniones.md](MySQL/joins-y-uniones.md): Cruces relacionales (`INNER`, `LEFT`, `RIGHT`), algoritmos de ejecucion (*Hash Join*) y operadores de conjuntos (`UNION` / `UNION ALL`).
  - [funciones-ventana-y-ctes.md](MySQL/funciones-ventana-y-ctes.md): CTEs (`WITH`), `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `PARTITION BY` y expresiones regulares (`REGEXP`).
  - [diseno-y-modelado/](MySQL/diseno-y-modelado/README.md): Módulo de Diseno Conceptual (DER), Cardinalidad (1:1, 1:N, N:M), Claves Primarias/Foraneas (B-Tree, UUIDv7, CASCADE) y Normalizacion rigurosa (1NF a 3NF/BCNF).
  - [procedimientos-almacenados-y-transacciones.md](MySQL/procedimientos-almacenados-y-transacciones.md): Procedimientos almacenados, `DELIMITER`, `LEAVE`/`ITERATE`, transacciones ACID, Undo/Redo log, niveles de aislamiento y bloqueos pesimistas (`FOR UPDATE`).
  - [vistas-y-triggers.md](MySQL/vistas-y-triggers.md): Vistas logicas con `WITH CHECK OPTION`, comparativa Vista vs CTE, emulacion de vistas materializadas y triggers con pseudorregistros `NEW`/`OLD`.
  - [SQL-Total-12-Dias/](MySQL/SQL-Total-12-Dias/README.md): Hub del curso, enlaces al certificado y plataformas de practica tecnica.
