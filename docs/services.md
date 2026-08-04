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