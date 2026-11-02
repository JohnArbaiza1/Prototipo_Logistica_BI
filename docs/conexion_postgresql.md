# Conexión de PostgreSQL con Power BI

Esta guía describe cómo habilitar un usuario de solo lectura para que Microsoft Power BI consulte el Data Warehouse del proyecto.

## 1. Arquitectura de conexión

Power BI debe conectarse a la base de datos `prototipo_logistica_dw`, específicamente al esquema `dw`. No necesita conectarse a la base operacional `prototipo_logistica` ni al esquema `staging`.

```text
OLTP ── ETL ──> PostgreSQL / prototipo_logistica_dw
                   ├── staging  (copias de trabajo del ETL)
                   └── dw       (modelo dimensional)  ──> Power BI Desktop
                                                          powerbi_reader
```

## 2. Requisitos previos

- PostgreSQL instalado y en ejecución.
- Base de datos `prototipo_logistica_dw` creada y poblada por el ETL.
- Esquema `dw` disponible.
- Rol `etl_writer` existente, utilizado por el proceso ETL para crear los objetos del DW.
- Acceso de superusuario o permisos suficientes para crear roles y conceder privilegios.
- Conectividad de red entre el equipo con Power BI y el servidor PostgreSQL.

## 3. Crear el usuario de Power BI

Se utiliza un rol independiente llamado `powerbi_reader`, con permisos de conexión y consulta, sin permisos para modificar los datos.

### Paso 1. Crear el rol y conceder conexión

Conectarse a la base `postgres` con un usuario administrador y ejecutar:

```sql
CREATE ROLE powerbi_reader
    LOGIN
    PASSWORD 'REEMPLAZAR_POR_UNA_CLAVE_SEGURA';

GRANT CONNECT ON DATABASE prototipo_logistica_dw
    TO powerbi_reader;
```

Reemplaza el valor de la contraseña por una clave segura. No utilices la contraseña de `etl_writer` ni de `etl_reader`.

### Paso 2. Conceder lectura sobre el esquema `dw`

Conectarse a `prototipo_logistica_dw` como administrador y ejecutar:

```sql
GRANT USAGE ON SCHEMA dw
    TO powerbi_reader;

GRANT SELECT ON ALL TABLES IN SCHEMA dw
    TO powerbi_reader;
```

Esto permite consultar las tablas existentes en el esquema `dw`. No se concede ningún permiso sobre `staging`.

### Paso 3. Conceder permisos predeterminados para futuras tablas

Ejecutar también en `prototipo_logistica_dw`:

```sql
ALTER DEFAULT PRIVILEGES FOR ROLE etl_writer
    IN SCHEMA dw
    GRANT SELECT ON TABLES TO powerbi_reader;
```

Este ajuste aplica a los objetos que `etl_writer` cree en el futuro dentro del esquema `dw`.

**Importante:** los privilegios predeterminados dependen del rol que crea los objetos. Esta configuración es correcta porque las migraciones y cargas crean los objetos del esquema `dw` como `etl_writer`. Si otro rol crea tablas, será necesario configurar sus privilegios predeterminados por separado.

Los permisos `SELECT` existentes se conservan cuando el ETL ejecuta `TRUNCATE` y vuelve a insertar registros.

## 4. Verificar los permisos

Conectarse como `powerbi_reader` a `prototipo_logistica_dw` y ejecutar:

```sql
SELECT * FROM dw.dim_tiempo LIMIT 5;        -- debe funcionar
SELECT * FROM staging.st_envios LIMIT 5;    -- debe dar error de permiso
```

También se pueden revisar los privilegios:

```sql
SELECT
    has_schema_privilege('powerbi_reader', 'dw', 'USAGE')            AS acceso_esquema_dw,
    has_schema_privilege('powerbi_reader', 'staging', 'USAGE')       AS acceso_esquema_staging,
    has_table_privilege('powerbi_reader', 'dw.dim_tiempo', 'SELECT') AS permiso_select,
    has_table_privilege('powerbi_reader', 'dw.dim_tiempo', 'INSERT') AS permiso_insert,
    has_table_privilege('powerbi_reader', 'dw.dim_tiempo', 'UPDATE') AS permiso_update,
    has_table_privilege('powerbi_reader', 'dw.dim_tiempo', 'DELETE') AS permiso_delete;
```

Resultado esperado:

| Comprobación | Resultado |
|---|---|
| `acceso_esquema_dw` | `true` |
| `acceso_esquema_staging` | `false` |
| `permiso_select` | `true` |
| `permiso_insert` | `false` |
| `permiso_update` | `false` |
| `permiso_delete` | `false` |

## 5. Configurar la conectividad de PostgreSQL

Si Power BI se ejecuta en otro equipo o en una máquina virtual, PostgreSQL debe aceptar conexiones desde ese equipo.

1. `listen_addresses` en `postgresql.conf` debe incluir la interfaz por la que llegará la conexión. No es recomendable abrir PostgreSQL a todas las interfaces sin necesidad.
2. `pg_hba.conf` debe incluir una regla `host` para la IP del equipo cliente, la base `prototipo_logistica_dw`, el usuario `powerbi_reader` y autenticación segura.
3. El firewall debe permitir el puerto de PostgreSQL (por defecto `5432`) únicamente desde los equipos autorizados.
4. Reinicia o recarga PostgreSQL después de modificar la configuración.

Ejemplo de regla para `pg_hba.conf` (reemplaza la IP por la del equipo que ejecuta Power BI):

```conf
host    prototipo_logistica_dw    powerbi_reader    IP_DEL_CLIENTE/32    scram-sha-256
```

No copies literalmente `IP_DEL_CLIENTE/32`. Para una red completa, utiliza el CIDR correspondiente y limita el rango tanto como sea posible.

> **Máquina virtual:** si Power BI corre en una máquina virtual con red NAT, el servidor PostgreSQL del equipo anfitrión suele verse con la dirección `10.0.2.2`.

## 6. Conectar desde Power BI Desktop

1. Abrir Power BI Desktop.
2. Seleccionar **Obtener datos**.
3. Buscar **Base de datos PostgreSQL**.
4. Introducir el servidor y la base de datos.

| Campo | Valor |
|---|---|
| Servidor | `HOST_O_IP_DEL_SERVIDOR:5432` |
| Base de datos | `prototipo_logistica_dw` |

5. Elegir el modo de conectividad **Importar**. Es el que usa el informe: copia los datos al modelo y los actualiza cuando se solicita.
6. Seleccionar autenticación mediante **Base de datos**.
7. Introducir `powerbi_reader` y su contraseña.
8. En el navegador, seleccionar las 13 tablas del esquema `dw`: 8 dimensiones, 4 hechos y `resultado_escenario`. Si solo aparece el esquema `dw` (y ninguna tabla de `staging`), los permisos están bien configurados.
9. Hacer clic en **Transformar datos** y revisar los tipos antes de cargar: claves como enteros, kilómetros y costos como decimales, `entregado_a_tiempo` como Verdadero/Falso y `fecha` como Fecha.
10. Hacer clic en **Cerrar y aplicar**.

## 7. Actualizar el informe

1. Ejecutar el ETL y comprobar que terminó correctamente y que el cuadre final no reporta diferencias.
2. Confirmar que `prototipo_logistica_dw` contiene la carga esperada.
3. Abrir el archivo PBIX.
4. Seleccionar **Actualizar** en la pestaña *Inicio*.
5. Comprobar que no existan errores de conexión o credenciales.

Los datos del modo Importar no se actualizan solos. Para actualizaciones programadas en Power BI Service puede ser necesario un gateway de datos local, según dónde esté alojado PostgreSQL.

## 8. Solución de problemas

| Problema | Qué revisar |
|---|---|
| *«El certificado remoto no es válido según el procedimiento de validación»* | El servidor usa un certificado que Windows no reconoce. En *Archivo → Opciones y configuración → Configuración de origen de datos → Editar permisos*, desmarcar **Cifrar conexiones**, guardar y actualizar |
| Pide instalar un proveedor | Instalar el controlador Npgsql y reiniciar Power BI |
| No se puede establecer conexión | Dirección del servidor, puerto, servicio PostgreSQL y firewall |
| Tiempo de espera agotado | `listen_addresses`, regla de `pg_hba.conf`, ruta de red y firewall |
| Error de autenticación | Usuario, contraseña y método de autenticación |
| Permiso denegado para el esquema | `GRANT USAGE ON SCHEMA dw` |
| Permiso denegado para una tabla | `GRANT SELECT ON ALL TABLES` y privilegios predeterminados del rol creador |
| No aparecen tablas | Confirmar que se seleccionó `prototipo_logistica_dw` y el esquema `dw` |
| `MapVisualNotEnabled` en el mapa | Activar los objetos visuales de mapa en *Archivo → Opciones y configuración → Opciones → Seguridad*, reiniciar Power BI y comprobar que hay internet |
| No se actualizan los datos | Comprobar que el ETL terminó correctamente y revisar las credenciales guardadas en Power BI |

## Consideraciones de seguridad

- No conectar Power BI como superusuario.
- No utilizar `etl_writer` ni `etl_reader` como usuario del dashboard: `etl_reader` tiene lectura sobre `staging` y `dw`, lo que contradice el aislamiento previsto.
- No conceder permisos de escritura a `powerbi_reader`.
- No exponer PostgreSQL directamente a Internet.
- No almacenar contraseñas en el repositorio ni en capturas de pantalla.