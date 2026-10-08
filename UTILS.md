**stack exclusivo** para PostgreSQL dentro de su propia carpeta.

* Arrancar / detener la base sin afectar tus apps.
* Versionar (o ignorar) el archivo `compose.yml` aparte de cada proyecto.
* Mantener el volumen y la red controlados en un solo lugar.

A continuación tienes los pasos detallados desde WSL.

---

## 1. Crea una carpeta para el stack de la BD

```bash
# Por ejemplo dentro de tu $HOME
mkdir -p ~/Projects/pg-db
cd ~/Projects/pg-db
```

---

## 2. Escribe `compose.yml`

```yaml
services:
  db_pg:
    image: postgres:15             # o postgres:16 si lo prefieres
    container_name: pg_db
    restart: always

    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      TZ: America/Managua

    volumes:
      - data:/var/lib/postgresql/data
    networks:
      - pg-network
    ports:
      - "5432:5432"                # expón solo si vas a conectarte desde el host

volumes:
  data:
    name: pg-data

networks:
  pg-network:
    external: true
```

* **`data`** guarda los datos persistentes.
* **`pg_network`** es la red que compartirán tus demás proyectos.

---

## 3. Arranca únicamente la base de datos

```bash
docker compose up -d --build           # dentro de ~/docker/pg-db
```

Verifica que todo está ok:

```bash
docker ps        # debería aparecer pg_central
docker network ls | grep pg-network
```

---

## 4. Conéctate y crea tu base “my_db”

### 4.1 Con `psql` instalado en tu WSL

```bash
psql -h localhost -U postgres
# ingresa la contraseña de postgres cuando la pida

postgres=# CREATE DATABASE my_db;
postgres=# \q
```
---


### Ventajas de este esquema

| Ventaja                    | Explicación                                                                     |
| -------------------------- | ------------------------------------------------------------------------------- |
| **Independencia total**    | Puedes `docker compose down` en cualquier app sin tumbar PostgreSQL.            |
| **Escalabilidad**          | Agrega tantos servicios cliente como quieras, basta que usen la red `pg-network`. |
| **Simplicidad de backup**  | Todo queda en el volumen `pg_data`; respáldalo o mapea `/backups`.              |

//////////////////////////////////////////////////////////////////////////////////////////////////////

| Acción                        | Comando                                    |
| ----------------------------- | ------------------------------------------ |
| Solo detener                  | `docker compose stop`    |
| Detener + eliminar contenedor | `docker compose down`    |
| Eliminar todo (incluye datos) | `docker compose down -v` |
| Volver a levantar             | `docker compose -d`      |
