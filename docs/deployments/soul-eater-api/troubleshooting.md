# Troubleshooting

This document covers the main problems I ran into while deploying the Soul Eater API and how I solved them.

## PostgreSQL Connection Failed

### Problem

The FastAPI backend could not connect to the PostgreSQL container.

### Cause

The database port in the Docker Compose environment variables was set incorrectly.

```text
DB_PORT=5342
```

PostgreSQL was actually listening on its default port:

```text
5432
```

### Solution

I updated the environment variable to:

```text
DB_PORT=5432
```

and recreated the backend container so it received the updated configuration. After the change, the backend was able to connect to PostgreSQL successfully.

---

## PostgreSQL JSONB Insert Error

### Problem

The database seeding script failed when inserting some of the Soul Eater data into PostgreSQL.

### Cause

Some fields, such as `occupations`, `partners`, and `abilities`, contained Python lists. These fields were stored as PostgreSQL `JSONB` columns, but the Python lists needed to be converted into JSON compatible values before being inserted.

### Solution

I used Psycopg's `Json` adapter and converted the list values when inserting them:

```python
from psycopg2.extras import Json

Json(character["occupations"])
Json(character["partners"])
Json(character["abilities"])
```

I also changed the SQL insert statements to only list the column names so that each value was inserted into the correct column. After making these changes, the database was seeded successfully.

## Frontend Could Not Access the Backend

### Problem

The frontend container loaded successfully, but API data did not appear in the application.

The browser console showed a CORS error.

### Cause

The frontend and backend were being accessed through different ports:

```text
Frontend ---> http://<VM-IP>:3000
Backend  ---> http://<VM-IP>:8000
```

The browser treats these as different origins and the homelab frontend URL was not included in the FastAPI CORS configuration.

### Solution

I added the homelab frontend URL to the FastAPI allowed origins. After rebuilding the backend container, the frontend was able to communicate with the API. I added Nginx later on as a reverse proxy so that both frontend and API requests could go through the same main entry point.

## Nginx Connection Reset

### Problem

After adding the Nginx reverse proxy, accessing:

```text
http://<VM-IP>:8080
```

failed with:

```text
Recv failure: Connection reset by peer
```

### Cause

Docker Compose mapped:

```text
VM :8080 -> Nginx container :80
```

but the Nginx configuration was accidentally set to:

```nginx
listen 90;
```

Docker was sending requests to port `80` inside the container while Nginx was listening on port `90`.

### Solution

I changed the Nginx configuration to:

```nginx
listen 80;
```

and restarted the Nginx container. I tested the deployment again using:

```bash
curl http://localhost:8080
```

and then accessed it from another computer on the local network. The application loaded successfully through Nginx.