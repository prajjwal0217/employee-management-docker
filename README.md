# Employee Management System --- Dockerized 3-Tier Application

A containerized employee management application built to practise
deploying and operating a three-tier web application with Docker and
Docker Compose. The project separates the frontend, backend API, and
database into independent containers and keeps the application and
database tiers on an internal Docker network.

**Repository:**
https://github.com/prajjwal0217/employee-management-docker

## Table of Contents

-   [Overview](#overview)
-   [Features](#features)
-   [Architecture](#architecture)
-   [Technology Stack](#technology-stack)
-   [Repository Structure](#repository-structure)
-   [Request Flow](#request-flow)
-   [Docker Networking](#docker-networking)
-   [Persistence](#persistence)
-   [Configuration and Secrets](#configuration-and-secrets)
-   [Prerequisites](#prerequisites)
-   [Run the Project](#run-the-project)
-   [Useful Docker Commands](#useful-docker-commands)
-   [Troubleshooting](#troubleshooting)
-   [Security Notes](#security-notes)
-   [Learning Outcomes](#learning-outcomes)
-   [Possible Improvements](#possible-improvements)

## Overview

This project packages an Employee Management System into three Docker
services:

1.  **Web tier:** Angular frontend served by Nginx.
2.  **Application tier:** Spring Boot REST API.
3.  **Database tier:** MySQL.

Docker Compose builds and runs the services together. Only the web tier
publishes ports to the host. The backend and database are reached
through Docker's internal networking rather than by exposing their ports
directly to the host.

The project is intended as a hands-on learning exercise in
containerization, multi-container networking, persistent storage,
reverse proxying, environment configuration, and application/database
troubleshooting.

## Features

-   Three-tier architecture with separate web, application, and database
    containers.
-   Custom Docker images for the frontend, backend, and database.
-   Nginx serves the frontend and acts as a reverse proxy to the backend
    API.
-   MySQL stores employee records in a named Docker volume.
-   Docker Compose manages service configuration and lifecycle.
-   Database credentials are supplied through local configuration and
    secret files.
-   A dedicated MySQL application account is used by Spring Boot instead
    of the database root account.
-   The frontend displays operation-result messages and clears the form
    after the relevant operation completes.

> Feature details reflect the project as configured during development.
> Confirm current implementation and endpoint names in the source code
> before relying on them in another environment.

## Architecture

``` mermaid
flowchart TB
    Browser[User Browser]
    subgraph DockerHost[Docker Host]
        subgraph WebTier[Web Tier]
            Nginx[Nginx / Angular Frontend]
        end
        subgraph InternalNetwork[Docker Internal Network]
            App[Spring Boot API]
            DB[(MySQL Database)]
        end
        Browser -->|HTTP :80 / HTTPS :443| Nginx
        Nginx -->|Reverse proxy to API| App
        App -->|JDBC / MySQL :3306| DB
    end
```

### Service responsibilities

  -----------------------------------------------------------------------
  Service           Container name    Responsibility    Host-published
                                                        ports
  ----------------- ----------------- ----------------- -----------------
  Web               `employee-web`    Serves the        `80`, `443`
                                      Angular frontend  
                                      and proxies API   
                                      requests through  
                                      Nginx             

  Application       `employee-app`    Runs the Spring   None
                                      Boot application  
                                      and handles API   
                                      requests          

  Database          `employee-db`     Stores            None
                                      application data  
                                      in MySQL          
  -----------------------------------------------------------------------

The application connects to MySQL using the Docker service/container DNS
name `employee-db` and port `3306`. It should not use `localhost` as the
database host because `localhost` inside the application container
refers to that same application container.

## Technology Stack

  Component                       Technology
  ------------------------------- ------------------------
  Frontend                        Angular
  Web server / reverse proxy      Nginx
  Backend                         Java, Spring Boot
  Database                        MySQL 8.x
  Containerization                Docker
  Multi-container orchestration   Docker Compose
  Database connectivity           JDBC / MySQL Connector
  Source control                  Git and GitHub

## Repository Structure

The repository uses separate build contexts for the web and application
images, along with a database image and a Compose file. The exact paths
may change as the project evolves.

``` text
employee-management-docker/
├── docker-compose.yml
├── Dockerfile                 # Database image build
├── secrets/                   # Local secret files; do not commit
├── environment.env            # Local application environment; do not commit
├── initdb/                    # Optional database initialization scripts
├── App/
│   └── Dockerfile             # Spring Boot application image
└── Web/
    └── Dockerfile             # Angular/Nginx web image
```

Check the actual repository tree for the current filenames and build
contexts before running commands. Generated JAR files, Angular build
output, local environment files, credentials, and private TLS keys
should not be committed.

## Request Flow

1.  A user opens the application in a browser through the host's
    published web port.
2.  Nginx serves the built Angular static files.
3.  When the frontend requests employee data or submits an operation, it
    sends an API request to the configured API path.
4.  Nginx forwards that request to the Spring Boot service over the
    Docker network.
5.  Spring Boot processes the request and uses JDBC to connect to MySQL.
6.  MySQL returns the result to Spring Boot.
7.  Spring Boot's response travels back through Nginx to the browser,
    where the frontend displays the result.

Using Nginx as a reverse proxy allows the browser to use a single public
origin for frontend and API traffic when the frontend uses relative API
paths. This can avoid the need for cross-origin CORS configuration in
that deployment arrangement.

## Docker Networking

The Compose project creates a network for service-to-service
communication.

-   The browser reaches Nginx through the published host ports.
-   Nginx reaches the backend using its Docker service name and
    application port.
-   The backend reaches MySQL at `employee-db:3306`.
-   The backend and database do not need host-published ports for this
    flow.

Inspect the active network and container port mappings:

``` bash
docker compose ps
docker network ls
docker inspect employee-app
docker port employee-web
```

To check whether a port is published, review `docker compose ps` or
`docker port <container-name>`. Do not publish the database port merely
to make the application connect to it.

## Persistence

The Compose configuration uses a named volume:

``` yaml
volumes:
  employee-db-data:
    driver: local
```

The database service mounts this volume at `/var/lib/mysql`.

This keeps MySQL data outside the container's writable layer. Recreating
the database container does not normally remove the named volume, so
records can persist across container recreation.

**Warning:** `docker compose down -v` removes Compose-managed named
volumes and can delete the database data. Do not run it unless you
intentionally want to remove the data and have a suitable backup.

## Configuration and Secrets

The project uses separate values for the database administrator and
application database account.

  ---------------------------------------------------------------------------------
  File / setting                      Purpose
  ----------------------------------- ---------------------------------------------
  `secrets/db_root_password.txt`      Password for MySQL's administrative `root`
                                      account

  `environment.env`                   Local application environment variables,
                                      including the application DB password in the
                                      current setup

  `DB_URL`                            JDBC URL, for example
                                      `jdbc:mysql://employee-db:3306/employee_db`

  `DB_USERNAME`                       Dedicated database user, for example
                                      `app_user`
  ---------------------------------------------------------------------------------

The current application image reads its database password from an
environment variable provided through `environment.env`. The image's
startup process was not successfully adapted to `_FILE`-style password
variables, so this project currently uses the working
environment-variable method.

### Important distinctions

-   Docker Compose `env_file` injects variables into the container
    environment. It does **not** mount that file as a Docker secret.
-   Compose secrets are mounted as files only into services that request
    them.
-   The official MySQL image's `MYSQL_*` initialization variables
    generally apply when its data directory is first initialized.
    Changing these variables does not automatically change users or
    passwords in an existing persistent database.
-   The application database account should have only the privileges
    required by the application; avoid using `root` for normal
    application queries.
-   Keep local credentials out of Git. `.gitignore` does not remove a
    secret that was committed in an earlier commit.

Before sharing or deploying the project, verify that `environment.env`
and files under `secrets/` are ignored by Git and are not tracked. For
production environments, prefer a managed secret store or an
orchestrator's secret-management facility.

## Prerequisites

Install the following on the host:

-   Docker Engine or Docker Desktop.
-   Docker Compose v2 (`docker compose`).
-   Git, if cloning the repository.

The custom images must be built from the repository's Dockerfiles.
Ensure required build artifacts (such as the Spring Boot JAR and Angular
build output, if copied by the Dockerfiles) are present at the paths
expected by those Dockerfiles.

## Run the Project

### 1. Clone the repository

``` bash
git clone https://github.com/prajjwal0217/employee-management-docker.git
cd employee-management-docker
```

### 2. Prepare local configuration

Create the secret directory and files expected by the Compose file:

``` text
secrets/
├── db_root_password.txt
└── app_db_password.txt
```

Use strong, unique passwords. Do not commit these files.

Create `environment.env` with the variables expected by the backend
image and application, for example:

``` dotenv
DB_PASSWORD=replace_with_your_application_password
```

Use the same password that is configured for the dedicated MySQL
application account. Do not use the example value as a real password.
Add any additional variables required by your actual application
configuration.

**Note:** The exact secret files required depend on the current
`docker-compose.yml`. If your Compose file does not declare
`app_db_password`, do not create it solely for this example; follow the
configuration actually checked into the repository.

### 3. Build and start the services

``` bash
docker compose config --quiet
docker compose up -d --build
```

### 4. Check service status

``` bash
docker compose ps
docker compose logs --tail=100
```

### 5. Open the application

-   HTTP: `http://localhost`
-   HTTPS: `https://localhost` if TLS is configured and the required
    certificates are present.

A self-signed certificate may trigger a browser security warning during
local testing. Do not treat a self-signed certificate as production
certificate management.

### 6. Stop the services

``` bash
docker compose down
```

This stops and removes the containers and default Compose network but
normally retains the named database volume.

## Useful Docker Commands

  ---------------------------------------------------------------------------------
  Task                                Command
  ----------------------------------- ---------------------------------------------
  Build and start                     `docker compose up -d --build`

  List services                       `docker compose ps`

  Follow all logs                     `docker compose logs -f`

  Follow one service's logs           `docker compose logs -f employee-app`

  Restart a service                   `docker compose restart employee-app`

  Rebuild a specific service          `docker compose up -d --build employee-app`

  Open a shell in a container         `docker exec -it employee-app sh`

  Inspect a container                 `docker inspect employee-app`

  List images                         `docker images`

  List volumes                        `docker volume ls`

  Stop/remove containers              `docker compose down`
  ---------------------------------------------------------------------------------

## Troubleshooting

### Application cannot connect to MySQL

1.  Confirm the database container is running: `docker compose ps`.
2.  Review database logs: `docker compose logs employee-db`.
3.  Verify that the JDBC URL uses `employee-db:3306`, not
    `localhost:3306`.
4.  Verify that the database name and application username are correct.
5.  Verify that the user's password matches the password configured in
    MySQL.
6.  Check the grants for the application account.
7.  Remember that initialization variables do not recreate users or
    reset passwords in an already-initialized volume.

### `Access denied for user`

This usually means MySQL was reached but rejected authentication or the
account's host/privileges did not match. Check the username, password,
account host, and grants. Avoid printing passwords in logs or command
output.

### Changes to initialization scripts do not run

The MySQL image runs scripts in `/docker-entrypoint-initdb.d/` during
first-time initialization of an empty data directory. Scripts are not
automatically rerun on every container restart. For an existing
database, execute the required SQL against that database or use a
controlled migration process.

### Frontend API request fails

1.  Check the browser's developer tools for the request URL and
    response.
2.  Review Nginx logs: `docker compose logs employee-web`.
3.  Review Spring Boot logs: `docker compose logs employee-app`.
4.  Confirm Nginx's `proxy_pass` target uses the backend's Docker
    service name and correct port.
5.  Confirm the frontend calls the expected API path.
6.  If the frontend and API use different browser origins, check CORS
    configuration. If Nginx proxies both through the same origin, prefer
    relative API paths where appropriate.

### Port is already in use

Check the host port mappings in `docker-compose.yml`. Stop the
conflicting service or change the host-side port, for example from
`8080:80` to `8088:80`, then open the corresponding host port in the
browser.

## Security Notes

-   Do not commit credentials, `.env`/environment files, secret files,
    private keys, or real certificates containing private key material.
-   Use a dedicated least-privilege database account for the
    application.
-   Keep MySQL and Spring Boot ports unpublished unless there is a
    specific, controlled administrative requirement.
-   Use HTTPS with a properly managed certificate for deployments beyond
    local testing.
-   Pin base image versions rather than relying on `latest` for
    repeatable builds.
-   Review container logs and `docker inspect` output before sharing
    them; they may expose environment configuration.
-   Back up persistent database data before making destructive changes.

## Learning Outcomes

This project provides practice with:

-   Building custom Docker images.
-   Running a multi-container application with Docker Compose.
-   Separating web, application, and database tiers.
-   Configuring Nginx as a reverse proxy.
-   Using Docker DNS names for service-to-service communication.
-   Persisting MySQL data with named volumes.
-   Configuring application database accounts and privileges.
-   Handling environment variables and secret files.
-   Troubleshooting connectivity, authentication, container startup, and
    frontend API requests.
-   Using Git and GitHub to document infrastructure configuration.

## Possible Improvements

-   Use a safer, repeatable secret-injection mechanism for the custom
    Spring Boot image.
-   Automate database account setup with a secure initialization or
    migration process.
-   Add health checks and dependency readiness handling.
-   Pin and regularly update base images.
-   Add automated build and smoke tests through GitHub Actions.
-   Add backup and restore instructions for the MySQL volume.
-   Add a network/architecture diagram and screenshots of the working
    application.
-   Document API endpoints and example requests.
-   Configure production TLS and deployment-specific secret management.