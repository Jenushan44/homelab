# Deployment

## Docker Compose 

Docker Compose manages the application services on `ubuntu-server-01` using YAML configuration files. 

## Nginx Test Service

### Configuration Location

```text
~/compose/nginx/compose.yaml
```

### Start Services

```bash
sudo docker compose up -d
```

### View Service Status

```bash
sudo docker compose ps
```

### View Logs

```bash
sudo docker compose logs
```

### Stop Services

```bash
sudo docker compose stop
```

### Start Existing Services

```bash
sudo docker compose start
```

### Restart Services

```bash
sudo docker compose restart
```

### Remove Containers and Network

```bash
sudo docker compose down
```

## PostgreSQL

### Start Services

```bash
sudo docker compose up -d
```

### Connect to PostgreSQL

```bash
sudo docker exec -it postgres psql -U admin -d soul_eater
```

### Verify Persistence

1. Create a table
2. Insert data
3. Run:

```bash
sudo docker compose down
sudo docker compose up -d
```

4. Verify the data still exists

## Soul Eater API Backend

### Build and Start

```bash
sudo docker compose up -d --build
```

### View Status

```bash
sudo docker compose ps
```

### View Logs

```bash
sudo docker compose logs soul_eater_api_backend
```

### Access Swagger UI

```text
http://<VM-IP>:8000/docs
```