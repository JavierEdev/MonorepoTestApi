# Monorepo de práctica DevOps

Aplicación de tareas con backend en Go y Gin, frontend en React y PostgreSQL.
La configuración disponible por ahora permite levantar la base de datos local;
el backend, el frontend y las tablas se añadirán en los siguientes pasos.

El objetivo es ejecutar todo localmente. Después de la aplicación funcional se
añadirán contenedores para backend y frontend, Kubernetes local con kind y
observabilidad con Prometheus y Grafana. El despliegue en nube queda como una
extensión futura. Estas etapas aún no están implementadas.

## Requisitos

- Docker con el motor en ejecución y Docker Compose disponible.
- DBeaver para explorar la base de datos visualmente.

Ejecuta los comandos siguientes desde la raíz del repositorio.

## Levantar PostgreSQL

1. Crea tu configuración local, si aún no tienes un archivo `.env`:

   ```bash
   cp -n .env.example .env
   ```

   `.env` está excluido de Git. `.env.example` contiene los valores de ejemplo
   para reproducir el entorno en otra máquina.

2. Valida la configuración:

   ```bash
   docker compose config --quiet
   ```

3. Descarga la imagen e inicia la base de datos, esperando a que esté lista:

   ```bash
   docker compose up -d --wait db
   ```

4. Consulta el estado:

   ```bash
   docker compose ps
   ```

   El servicio `db` debe aparecer como `healthy`.

## Conectar DBeaver

Crea una conexión de tipo **PostgreSQL** e introduce los valores de `.env`.
Con la configuración de ejemplo:

| Campo | Valor |
| --- | --- |
| Host | `localhost` |
| Puerto | `5432` |
| Base de datos | `taskapp` |
| Usuario | `taskapp` |
| Contraseña | `taskapp_local` |

Pulsa **Test Connection**. Si DBeaver solicita descargar el controlador de
PostgreSQL, permite la descarga. Guarda la conexión cuando la prueba funcione.

Para verificarla, abre un editor SQL y ejecuta:

```sql
SELECT current_database(), current_user, version();
```

La base estará vacía inicialmente. Las tablas y sus relaciones se crearán
mediante migraciones SQL versionadas en el repositorio.

## Qué configura Compose

- **`db`:** PostgreSQL 18 desde la imagen oficial.
- **`environment`:** base, usuario y contraseña iniciales tomados de `.env`.
- **`ports`:** publica PostgreSQL en `127.0.0.1:5432`, accesible desde tu Mac.
  Si el puerto está ocupado, cambia `POSTGRES_PORT` en `.env` y usa ese puerto
  también en DBeaver.
- **`postgres_data`:** volumen que conserva los datos al recrear el contenedor.
- **`healthcheck`:** comprueba periódicamente que PostgreSQL acepta conexiones.
  La comprobación de autenticación se realiza conectándose con DBeaver o SQL.

El volumen se monta en `/var/lib/postgresql`, la ubicación indicada para
PostgreSQL 18 en la [documentación de la imagen oficial](https://github.com/docker-library/docs/blob/master/postgres/README.md).

Las variables de inicialización solo se aplican cuando el volumen está vacío.
Cambiar usuario, contraseña o base en `.env` después del primer arranque no
modifica automáticamente una base existente; esos cambios requieren SQL de
administración.

## Comandos útiles

Ver los logs:

```bash
docker compose logs -f db
```

Abrir una sesión SQL dentro del contenedor:

```bash
docker compose exec db sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"'
```

Detener y eliminar el contenedor conservando los datos:

```bash
docker compose down
```

Para volver a iniciarlo, usa `docker compose up -d --wait db`. Los datos se
conservan mientras no se elimine el volumen `postgres_data`.

## Convención de pull requests

El título de cada PR debe empezar con `[MR-NUMERO]`, con `MR` en mayúsculas y
uno o más dígitos. Ejemplo: `[MR-123] Agregar configuración de PostgreSQL`.
El identificador no tiene que coincidir con el número del PR en GitHub.

La [GitHub App PRLint](https://github.com/apps/prlint) valida el título usando
las reglas de `.github/prlint.json`. Instala o habilita la app para este
repositorio. Si la validación falla, corrige el título del PR.

Para impedir merges cuando falle, configura el check que publique PRLint como
obligatorio en las reglas de protección de la rama de destino en GitHub.
