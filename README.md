# Power BI – Prototipo Logística

Este repositorio contiene el archivo de Power BI del proyecto Prototipo Logística, desarrollado para visualizar y analizar la información procesada por el pipeline ETL.

El dashboard obtiene sus datos desde una base de datos PostgreSQL correspondiente al Data Warehouse (DW), construido a partir de la base de datos operacional (OLTP) mediante un proceso ETL.

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