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