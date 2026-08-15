# Services

## ubuntu-server-01

### Purpose

Main application host for the homelab. 

### Operating System

Ubuntu Server 24.04 LTS

### Responsibilities

- Host Docker
- Host personal projects
- Host databases
- Run reverse proxy

### Current Status

|        Service         |  Status   |
|------------------------|-----------|
| Docker                 | Installed |
| Docker Compose         | Installed |
| PostgreSQL             | Active    |
| Nginx                  | Active    |
| Soul Eater API         | Planned   |
| API Security Analyzer  | Planned   |
| Portfolio              | Planned   |

## Docker 

### Purpose

Runs and manages the applications hosted on `ubuntu-server-01`

### Status

Installed

### Responsibilities

- Run containerized applications
- Isolate application environments
- Manage application networking

## Nginx 

### Purpose 

Acts as the reverse proxy and entry point for applications hosted on `ubuntu-server-01`.

### Status

Active

### Responsibilities

- Recieve incoming HTTP/HTTPs requests
- Route requests to the correct application

## PostgreSQL

### Purpose

Provides persistent relational database storage for applications hosted on `ubuntu-server-01`.

### Status

Active

### Responsibilities

- Store application data
- Manage relational databases
- Persist data using Docker volumes

### Configuration

- Database: `soul_eater`
- User: `admin`
- Persistent Volume: `postgres_data`

## Soul Eater API Backend

### Purpose

Runs the FastAPI backend for the Soul Eater API.

### Status

In Progress

### Configuration

- Container: `soul_eater_api_backend`
- Internal Port: `8000`
- Host Port: `8000`
- Database Service: `postgres`

### Current Progress

- Backend Dockerfile created
- Docker image successfully built
- Backend container deployed with Docker Compose
- Swagger UI accessible from the local network

### Remaining Work

- Seed PostgreSQL with the existing API data
- Update routes to query PostgreSQL
- Verify database-backed endpoints