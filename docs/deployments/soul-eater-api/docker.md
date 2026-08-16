# Docker Deployment

## Overview

Docker is used to run the different parts of the Soul Eater API in separate containers. The frontend and backend each use their own Dockerfile to build an image containing the code, runtime, libraries, and dependencies needed to run that part of the application. PostgreSQL and Nginx use existing Docker images. Docker Compose is used to configure and manage all four of the containers together.

## Containers

The deployment uses four containers:

- **Frontend container** - runs the Next.js frontend.
- **Backend container** - runs the FastAPI backend.
- **PostgreSQL container** - runs the PostgreSQL database.
- **Nginx container** - runs the reverse proxy.

## Docker Flow

```text
Dockerfile ---> Docker Image ---> Container
```

A Dockerfile has the instructions used to build an image. The image has the packaged application environment, including the runtime, dependencies and application code. Docker then creates a running container from that image. For the Soul Eater API, separate images are created for the frontend and backend because they use different runtimes and dependencies.

## Docker Compose

Docker Compose is used to manage all of the containers together.

The `compose.yaml` file defines things such as:

- which image or Dockerfile each service uses
- ports
- environment variables
- volumes
- dependencies between services

Running:

```bash
sudo docker compose up -d --build
```

builds any of the required images and starts the containers. This makes it possible to manage the full application stack with one configuration file instead of starting and configuring every container separately.