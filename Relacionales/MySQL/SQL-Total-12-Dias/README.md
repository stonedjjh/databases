# SQL Total - Domina Bases de Datos de 0 a Avanzado en 12 Dias

![Status](https://img.shields.io/badge/Status-Completado-brightgreen)
![Score](https://img.shields.io/badge/Evaluaci%C3%B3n-168%20%7C%20Avanzado-blueviolet)
![Engine](https://img.shields.io/badge/Engine-MySQL%208.0+-4479A1?style=flat&logo=mysql&logoColor=white)
![Certificate](https://img.shields.io/badge/Certificado-Udemy-blue?logo=udemy)

Apuntes tecnicos avanzados, diseno de arquitectura relacional y recopilacion de practicas correspondientes al programa intensivo **SQL Total (12 Dias)**.

---

## Certificacion y Evaluacion de Habilidades

- **Certificado Oficial de Finalizacion:** [Verificar en Udemy (ID: UC-3705a432-c742-43f1-832c-0c7b30ade390)](https://www.udemy.com/certificate/UC-3705a432-c742-43f1-832c-0c7b30ade390/)
- **Copia del Certificado:** [SQL Total.jpg](../../docs/SQL%20Total.jpg)
- **Evaluacion de Habilidades (Lenguaje de Consulta Estructurado - SQL):**
  - **Puntaje Obtenido:** **168 / 200 (Nivel Avanzado)**.
  - **Rendimiento:** Superior al 78% de los estudiantes evaluados en la plataforma.
  - **Comprobante:** [evaluacion.png](../../docs/evaluacion.png)

---

## Guias Tecnicas y Modulos Construidos

Los apuntes superan el temario elemental del curso, profundizando en optimizacion de motores, estructuras B-Tree de InnoDB, arquitectura relacional y patrones profesionales:

1. **[Anatomia del Almacenamiento en MySQL](../anatomia-del-almacenamiento.md)**: Mapeo de bases de datos y tablas a archivos de disco (`.ibd`), tablespaces y configuracion del motor InnoDB.
2. **[Comandos y Funciones del Motor](../comandos.md)**: Referencia rapida de variables del sistema (`@@datadir`, `@@lower_case_table_names`), diagnostico y funciones de auditoria (`NOW()`, `CURRENT_USER()`).
3. **[Taxonomia del Lenguaje SQL: DDL, DML, DQL, TCL y DCL](../lenguaje-sql-ddl-dml-dcl-tcl.md)**: Diferencias entre `DROP`, `TRUNCATE` y `DELETE`, riesgos de auto-commit implicito en DDL, transacciones ACID y principio de menor privilegio con DCL.
4. **[Consultas en MySQL (DML, Subconsultas y Funciones)](../consultas.md)**: Proyecciones, filtros, agrupaciones, subconsultas, `IF`/`CASE`, `CAST` y funciones nativas.
5. **[Cruces de Tablas y Uniones (JOINs y UNION)](../joins-y-uniones.md)**: Algebra relacional, `INNER`, `LEFT`, `RIGHT JOIN`, algoritmos de ejecucion (*Hash Join* en MySQL 8.0) y operadores de conjuntos (`UNION` vs `UNION ALL`).
6. **[Funciones de Ventana, CTEs y REGEXP](../funciones-ventana-y-ctes.md)**: `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `PARTITION BY`, `OVER()`, expresiones de tabla comunes (`WITH`) y expresiones regulares.
7. **[Modulo Especializado: Diseno, Modelado y Normalizacion](../diseno-y-modelado/README.md)**:
   - [01. Diagrama Entidad-Relacion (DER)](../diseno-y-modelado/01-diagrama-entidad-relacion.md): Capa conceptual, entidades fuertes/debiles y sintaxis Mermaid.
   - [02. Cardinalidad y Tipos de Relaciones](../diseno-y-modelado/02-cardinalidad-y-relaciones.md): Mecanica 1:1, 1:N y tablas asociativas para N:M.
   - [03. Claves Primarias, Foraneas e Integridad](../diseno-y-modelado/03-claves-primarias-y-foraneas.md): `AUTO_INCREMENT` vs `UUIDv7`, indices agrupados y politicas `CASCADE` / `RESTRICT`.
   - [04. Normalizacion, Formas Normales y Anomalias](../diseno-y-modelado/04-normalizacion-y-anomalias.md): 1NF a 3NF/BCNF, descomposicion de dependencias y criterios de desnormalizacion (OLTP vs OLAP).
8. **[Procedimientos Almacenados, Transacciones ACID y Concurrencia](../procedimientos-almacenados-y-transacciones.md)**: Parametros `IN`/`OUT`/`INOUT`, etiquetas de control (`LEAVE`/`ITERATE`), `HANDLERS` para `SQLEXCEPTION`, niveles de aislamiento, bloqueos pesimistas (`FOR UPDATE`) y prevencion de deadlocks.
9. **[Vistas, Vistas Materializadas y Triggers](../vistas-y-triggers.md)**: Algoritmos `MERGE` vs `TEMPTABLE`, vistas actualizables con `WITH CHECK OPTION`, comparativa Vista vs CTE, emulacion de vistas materializadas y triggers `BEFORE`/`AFTER`.

---

## Plataformas de Practica y Resolucion de Problemas

Para continuar consolidando el nivel avanzado mediante la resolucion de retos algoritmicos y consultas de negocio:

- **[LeetCode (Database Problems)](https://leetcode.com/problemset/database/)**: Casos clasicos de entrevistas tecnicas (N-th Highest Salary, Consecutive Numbers, Department Top Three Salaries).
- **[HackerRank (SQL Practice)](https://www.hackerrank.com/domains/sql)**: Desafios por dificultad y sublenguajes (Basic Select, Advanced Select, Aggregations, Basic Join, Alternative Queries).
