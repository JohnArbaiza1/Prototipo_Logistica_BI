# 🚚 Prototipo Logística | Business Intelligence

<div align="center">

**Análisis y visualización del desempeño de las operaciones logísticas con Power BI y PostgreSQL ETL.**

[![Power BI](https://img.shields.io/badge/Power%20BI-Dashboard-F2C811?logo=powerbi&logoColor=black)](https://powerbi.microsoft.com/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Data%20Warehouse-4169E1?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Business Intelligence](https://img.shields.io/badge/Business-Intelligence-243B53)]()
[![Status](https://img.shields.io/badge/Estado-Prototipo-0A7B83)]()

</div>

---

## 📌 Descripción

**Prototipo Logística BI** es la capa de análisis y visualización de un proyecto orientado al seguimiento de las operaciones logísticas. El informe fue desarrollado en Microsoft Power BI y consume información almacenada en un Data Warehouse (DW) de PostgreSQL.

Los datos se originan en una base operacional (OLTP), son procesados mediante un pipeline ETL y posteriormente organizados en un modelo dimensional para facilitar su análisis.

Este repositorio contiene el archivo de Power BI y la documentación necesaria para preparar el acceso de lectura al Data Warehouse.

# 1. Crear el usuario de acceso para Power BI
Por seguridad, Power BI utilizará un usuario independiente con permisos exclusivamente de lectura.

Este usuario no podrá modificar, eliminar ni insertar información en el Data Warehouse.

## Paso 1. Crear el rol

Conectarse a PostgreSQL mediante un usuario superusuario, utilizando la base de datos postgres, y ejecutar:
``` SQL
CREATE ROLE powerbi_reader
LOGIN
PASSWORD 'una_clave_segura';

GRANT CONNECT ON DATABASE prototipo_logistica_dw
TO powerbi_reader;
```

Importante: Reemplace una_clave_segura por una contraseña segura.

## Paso 2. Otorgar permisos de lectura
Conectarse a la base de datos prototipo_logistica_dw como superusuario y ejecutar:

``` SQL
GRANT USAGE ON SCHEMA dw
TO powerbi_reader;

GRANT SELECT ON ALL TABLES IN SCHEMA dw
TO powerbi_reader;

ALTER DEFAULT PRIVILEGES FOR ROLE etl_writer
IN SCHEMA dw
GRANT SELECT ON TABLES TO powerbi_reader;
```

**Estos permisos permiten:**

- Acceder al esquema dw.
- Consultar las tablas existentes del Data Warehouse.
- Obtener permisos de lectura automáticamente sobre las nuevas tablas creadas por etl_writer dentro del esquema dw.

Nota: Los permisos se mantienen aunque el ETL trunque y reconstruya las tablas de hechos durante sus ejecuciones.