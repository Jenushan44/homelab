# PostgreSQL Database

## Overview

PostgreSQL is used as the database for the homelab deployment of the Soul Eater API. It runs in its own Docker container and stores the character, weapon, ability, organization, and arc data used by the FastAPI backend. The original Soul Eater API stores this data directly in Python files. For the homelab deployment, I added PostgreSQL so I could practice running a database as part of a Docker deployment and have the backend retrieve its data from a separate database service. The FastAPI backend connects to the PostgreSQL container using database settings passed through environment variables.

```text
FastAPI Backend Container
          |
          |-- Database Connection
          |----> PostgreSQL Container
                        |
                        |          
                        |--> Soul Eater Database
                                    |
                                    |-- characters
                                    |-- weapons
                                    |-- abilities
                                    |-- organizations
                                    |-- arcs
```

## PostgreSQL Container

PostgreSQL runs in its own Docker container using the PostgreSQL Docker image meaning I did not need to create a custom Dockerfile for PostgreSQL like the frontend and backend. Docker can use the PostgreSQL image to create and run the database container.

Docker Compose is used to configure the database container and provide values such as:

- database name
- database user
- database password
- PostgreSQL port
- persistent storage

PostgreSQL uses port `5432` inside the Docker environment.

The backend and PostgreSQL containers are apart of the same Docker Compose network. This allows the backend to connect to the database using the PostgreSQL service name instead of needing the container's IP address.

For example, the backend receives:

```text
DB_HOST = postgres
DB_PORT = 5432
```

This means the backend can use `postgres` as the hostname when creating the database connection.

## Database Tables

The database contains five main tables that match the types of data provided by the Soul Eater API:

```text
characters
weapons
abilities
organizations
arcs
```

Each table stores the data that was stored in the Python data files. For example, the `characters` table contains fields such as:

```text
id
name
role
affiliation
description
species
sex
soul_type
status
occupations
partners
abilities
debut
continuity
image_url
```

The other tables have the same general structure as their Pydantic models in the FastAPI application. This allowed the API responses to stay mostly the same while changing where the data comes from.

## JSONB Columns

Some of the API data has lists instead of individual values.

For example, a character can have multiple:

- occupations
- partners
- abilities

A weapon can also have multiple meisters and abilities, and an arc can contain multiple main characters. These values were stored using PostgreSQL `JSONB` columns.

For example:

```text
occupations -> JSONB
partners    -> JSONB
abilities   -> JSONB
```

This allows PostgreSQL to store the Python lists in a JSON format instead of converting the entire list into a normal text string. When inserting these values with Psycopg, I used `Json()` to convert the Python lists into values PostgreSQL could store in the JSONB columns.

For example:

```python
Json(character["occupations"])
Json(character["partners"])
Json(character["abilities"])
```

## Seeding the Database

The Soul Eater data already existed in the original Python data files, so I needed a way to copy that data into PostgreSQL. I created a database seeding script that imports the existing data:

```python
from app.data import characters, weapons, abilities, organizations, arcs
```

Then, the script connects to PostgreSQL and loops through each dataset. For example, the character data is inserted by looping through the existing character list:

```python
for character in characters:
    cursor.execute(...)
```

The same process is used for:

```text
characters
weapons
abilities
organizations
arcs
```

After all of the records are inserted, the changes are saved using:

```python
connection.commit()
```

The database cursor and connection are then closed.

```python
cursor.close()
connection.close()
```

This allowed me to reuse the data from the original API instead of manually entering every record into PostgreSQL.

## Connecting FastAPI to PostgreSQL

The FastAPI backend uses `psycopg2` to communicate with PostgreSQL. The database connection settings are read from environment variables:

```python
db_host = os.getenv("DB_HOST")
db_port = os.getenv("DB_PORT")
db_name = os.getenv("DB_NAME")
db_user = os.getenv("DB_USER")
db_password = os.getenv("DB_PASSWORD")
```

Then a database connection is created using these values:

```python
def get_connection():
    connection = psycopg2.connect(
        host=db_host,
        port=db_port,
        database=db_name,
        user=db_user,
        password=db_password
    )

    return connection
```

Using environment variables means the database credentials and connection information do not need to be hardcoded directly into the Python code. It also allows the same backend code to work differently depending on where it is deployed.

## Retrieving Data

Functions were added to get each type of data from PostgreSQL.

For example, the character database function runs:

```sql
SELECT * FROM characters;
```

The flow is:

```text
API Request --> FastAPI Route --> Database Function --> PostgreSQL Query --> Database Rows --> FastAPI Response
```

`RealDictCursor` is used when retrieving the rows:

```python
cursor = connection.cursor(cursor_factory=RealDictCursor)
```

Normally, Psycopg can return database rows as tuples.

For example:

```text
(1, "Maka Albarn", "Meister", ...)
```

Using `RealDictCursor` returns rows using their column names:

```text
{
    "id": 1,
    "name": "Maka Albarn",
    "role": "Meister"
}
```

This format works better with the FastAPI code because the original Python data was already structured using dictionaries. After the query finishes, the cursor and the connection are closed:

```python
cursor.close()
connection.close()
```

## Keeping the Public Deployment Working

The Soul Eater API was already deployed publicly before I created the homelab deployment. I wanted the homelab version to use PostgreSQL without affecting the public deployment. The API checks whether or not the `DB_HOST` environment variable exists before defaulting to the public deployment.

For example:

```python
if os.getenv("DB_HOST"):
    result = get_characters_from_db()
else:
    result = characters
```

When the application runs inside the homelab Docker environment, `DB_HOST` is given by Docker Compose. Then the backend gets its data from PostgreSQL:

```text
Homelab --> DB_HOST exists --> PostgreSQL
```

If the database environment variable is not available, the application continues using the original Python data:

```text
Public Deployment --> No DB_HOST --> Original Python Data
```

This allowed the homelab deployment and the public deployment to use the same codebase without requiring the public version to depend on the PostgreSQL container running in my home server.

## Persistent Storage

The PostgreSQL database needs to keep its data even if the PostgreSQL container is stopped, removed, or recreated so docker containers should not be treated as permanent storage. In order to solve this, the PostgreSQL container uses a Docker volume.

```text
PostgreSQL Container -- > postgres_data Volume --> Database Files
```

The PostgreSQL container runs the database software and the Docker volume stores the actual database files separately from the container. This means that the container can be recreated while the database data stays available. This is important for a database because losing a container should not mean losing all of the stored data.

## Testing

After setting up PostgreSQL, I tested the database separately before relying on it for the full application. I connected to PostgreSQL and checked that the tables had been created correctly and  I also checked the table structure to make sure that the expected columns and data types were there. After running the seeding script, I queried the tables to confirm that the Soul Eater data had been inserted. Then, I tested the API endpoints through FastAPI's Swagger UI.

I tested endpoints such as to confirm that the API could get the data from PostgreSQL and return it correctly:

```text
GET /characters
GET /characters/{id}

GET /weapons
GET /weapons/{id}

GET /abilities
GET /abilities/{id}

GET /organizations
GET /organizations/{id}

GET /arcs
GET /arcs/{id}
```

## Persistence Test

I also tested the Docker volume to make sure that the PostgreSQL data stayed available after the database container was recreated. This confirmed that the data was stored in the persistent Docker volume instead of depending on the life of the PostgreSQL container.

```text
                  psycopg2                           reads/writes
FastAPI Backend ------------> PostgreSQL Container ----------------> Persistent Docker Volume
```

The PostgreSQL container is responsible for running the database server and the Docker volume is responsible for keeping the database files persistent.