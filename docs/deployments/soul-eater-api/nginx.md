# Nginx Reverse Proxy

## Overview

Nginx is used as a reverse proxy for the Soul Eater API deployment. Before adding Nginx, the frontend and the backend were accessed separately using different ports:

```text
Frontend -> :3000
Backend  -> :8000
```

Nginx provides one main entry point and forwards the incoming requests to the correct container. The application can be accessed through port `8080` and Nginx handles communication with the frontend and backend containers.

```text
Client 
  |
  | HTTP :8080
  |
  | ---------> Nginx Container
                    |
                    |
                    |-- / --------> Frontend Container :3000
                    |
                    |-- /api/ ----> Backend Container :8000
```

## Reverse Proxy

A reverse proxy is between a client and the services running behind it. Instead of the client directly accessing each service, the client sends requests to Nginx. Nginx looks at the request and decides which container should receive it.

For the Soul Eater deployment:

```text
/      ---> Frontend
/api/  ---> Backend
```

This means the client only needs to access Nginx instead of directly accessing the frontend and backend on separate ports.

## Nginx Container

Nginx runs in its own Docker container and the deployment uses the Nginx Docker image, so a custom Dockerfile was not needed.

The Nginx service uses:

```yaml
image: nginx:latest
```

Docker creates the Nginx container from this image and the custom Nginx configuration is mounted into the container so that Nginx knows where requests should be sent.

## Nginx Configuration

The reverse proxy configuration uses two main routes:

```nginx
server {
    listen 80;

    location / {
        proxy_pass http://soul_eater_api_frontend:3000;
    }

    location /api/ {
        proxy_pass http://soul_eater_api_backend:8000/;
    }
}
```

Nginx listens on port `80` inside of its container. The Docker Compose configuration maps port `8080` on the Ubuntu Server VM to port `80` inside the Nginx container.

```text
Ubuntu VM :8080 ---> Nginx Container :80
```

This allows the application to be accessed using:

```text
http://<VM-IP>:8080
```

## Frontend Routing

The first Nginx route is:

```nginx
location / {
    proxy_pass http://soul_eater_api_frontend:3000;
}
```

Requests to `/` are forwarded to the frontend container.

For example:

```text
Client
  |
  | GET /
  |
  |---> Nginx
           |
           |---> Frontend Container :3000
```

The frontend container runs the Next.js application on port `3000`.

## Backend Routing

The second route is:

```nginx
location /api/ {
    proxy_pass http://soul_eater_api_backend:8000/;
}
```

Requests beginning with `/api/` are forwarded to the FastAPI backend.

For example:

```text
GET /api/characters
        |
        |
        |---> Nginx
                  |
                  |
                  |---> FastAPI Backend :8000
                                |
                                |
                                |---> /characters
```

The backend can then get the requested information from PostgreSQL and return the response.

## Docker Networking

Nginx, the frontend and the backend are connected to the same Docker Compose network. This allows Nginx to communicate with the other containers using their Docker Compose service names:

```text
soul_eater_api_frontend
soul_eater_api_backend
```

Nginx does not need to know the internal IP address of either container.

For example:

```nginx
proxy_pass http://soul_eater_api_frontend:3000;
```

Docker resolves `soul_eater_api_frontend` to the frontend container on the Compose network and the same process is used for the backend.

## Why Nginx Was Added

Before Nginx, the frontend and backend were accessed separately:

```text
Client
  |
  |-- :3000 --> Frontend
  |
  |-- :8000 --> Backend
```

After adding Nginx:

```text
Client
  |
  | :8080
  |
  |---> Nginx
          |
          |-- / -------> Frontend
          |
          |-- /api/ ---> Backend
```

This gives the application one entry point and keeps the routing between the frontend and backend behind Nginx. It also allowed the frontend to send API requests through the same Nginx entry point instead of directly accessing the backend through port `8000`.

## Frontend API URL

The homelab frontend was configured to send API requests through Nginx.

The `NEXT_PUBLIC_API_URL` was changed to use:

```text
http://<VM-IP>:8080/api
```

For example, when the frontend requests characters, the request first reaches Nginx:

```text
http://<VM-IP>:8080/api/characters
```

Nginx sees `/api/` and forwards the request to the FastAPI backend.

```text
Browser
   |
   | /api/characters
   |
   |---> Nginx :8080
              |
              |
              |---> FastAPI :8000
                         |
                         |
                         |---> PostgreSQL :5432
```

This also avoids having the browser access the frontend and backend through separate origins.

## Testing

After configuring Nginx, I tested it from the Ubuntu Server VM using:

```bash
curl http://localhost:8080
```

This confirmed that Nginx could receive a request through port `8080` and forward it to the frontend. Then, I accessed the application from another computer on the local network using:

```text
http://<VM-IP>:8080
```

The frontend loaded successfully and I was able to get the API data through the `/api/` route.

This confirmed that the complete request flow was working:

```text
Client
   |
   |
   |---> Nginx
          |
          |---> Frontend
          |
          |---> FastAPI
                  |
                  |
                  |---> PostgreSQL
```

Nginx is now the main entry point to the Soul Eater application running inside the homelab.