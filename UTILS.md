**stack exclusivo** para PostgreSQL dentro de su propia carpeta.

* Arrancar / detener la base sin afectar tus apps.
* Versionar (o ignorar) el archivo `db-compose.yml` aparte de cada proyecto.
* Mantener el volumen y la red controlados en un solo lugar.

A continuación tienes los pasos detallados desde WSL.

---

## 1. Crea una carpeta para el stack de la BD

```bash
# Por ejemplo dentro de tu $HOME
mkdir -p ~/docker/pg_central
cd ~/docker/pg_central
```

---

## 2. Escribe `db-compose.yml`

> **No** incluyas `POSTGRES_DB` si prefieres crear las bases manualmente; el contenedor arrancará sólo con las BD predeterminadas (`postgres`, `template0`, `template1`).

```yaml
# ~/docker/pg_central/db-compose.yml
version: "3.9"

services:
  db:
    image: postgres:15             # o postgres:16 si lo prefieres
    container_name: pg_central
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: pgadmin4
      # Sin POSTGRES_DB ➜ sólo se crea "postgres"
    volumes:
      - pg_data:/var/lib/postgresql/data
    networks:
      - datos_pg
    ports:
      - "5432:5432"                # expón solo si vas a conectarte desde el host

volumes:
  pg_data:

networks:
  datos_pg:
    name: datos_pg                 # red estable; se crea si no existe
```

* **`pg_data`** guarda los datos persistentes.
* **`datos_pg`** es la red que compartirán tus demás proyectos.

---

## 3. Arranca únicamente la base de datos

```bash
docker compose -f db-compose.yml up -d            # dentro de ~/docker/pg_central
```

Verifica que todo está ok:

```bash
docker ps        # debería aparecer pg_central
docker network ls | grep datos_pg
```

---

## 4. Conéctate y crea tu base “gstraffic”

### 4.1 Con `psql` instalado en tu WSL

```bash
psql -h localhost -U postgres
# ingresa la contraseña pgadmin4 cuando la pida

postgres=# CREATE DATABASE gstraffic;
postgres=# \q
```

*(Usa `createdb -h localhost -U postgres gstraffic` si prefieres).*

### 4.2 Sin `psql` en el host

Ejecuta `psql` dentro del propio contenedor:

```bash
docker exec -it pg_central psql -U postgres -d postgres -c "CREATE DATABASE gstraffic;"
```

---

## 5. Conecta tus demás proyectos

En cada `docker-compose.yml` de tus aplicaciones:

```yaml
services:
  web:
    ...
    environment:
      - DATABASE_URL=postgresql://postgres:pgadmin4@pg_central:5432/gstraffic
    networks:
      - datos_pg                 # ← misma red

networks:
  datos_pg:
    external: true               # solo la usa
```

*El bloque `db:` no se incluye, porque la BD ya existe.*

---

### Ventajas de este esquema

| Ventaja                    | Explicación                                                                     |
| -------------------------- | ------------------------------------------------------------------------------- |
| **Independencia total**    | Puedes `docker compose down` en cualquier app sin tumbar PostgreSQL.            |
| **Escalabilidad**          | Agrega tantos servicios cliente como quieras, basta que usen la red `datos_pg`. |
| **Simplicidad de backup**  | Todo queda en el volumen `pg_data`; respáldalo o mapea `/backups`.              |
| **Sin variables “basura”** | Al no definir `POSTGRES_DB`, tú decides qué bases crear y cómo llamarlas.       |





//////////////////////////////////////////////////////////////////////////////////////////////////////



| Acción                        | Comando                                    |
| ----------------------------- | ------------------------------------------ |
| Solo detener                  | `docker compose -f db-compose.yml stop`    |
| Detener + eliminar contenedor | `docker compose -f db-compose.yml down`    |
| Eliminar todo (incluye datos) | `docker compose -f db-compose.yml down -v` |
| Volver a levantar             | `docker compose -f db-compose.yml up -d`   |
