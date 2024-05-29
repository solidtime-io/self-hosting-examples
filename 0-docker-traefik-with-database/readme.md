# Example: Docker + Traefik + Database

This example shows how to run solidtime with Docker, Traefik, and with a database inside the same docker compose.

## Prerequisites

- [Docker](https://docs.docker.com/engine/install/)
- [Traefik V2 (Docker)](https://doc.traefik.io/traefik/getting-started/install-traefik/) with [Configuration Discovery](https://doc.traefik.io/traefik/providers/docker/)

If you don't know how to set up Traefik with automatic configuration discovery via labels, you can use [this guide](https://github.com/korridor/reverse-proxy-docker-traefik).

## Installation

1. Copy the example files and adjust the environment variables to your needs

```bash
cp laravel.env.example laravel.env
cp .env.example .env
```

2. (Only with Linux/Docker Engine) Correct the permissions of the `app-storage` and `logs` directories.

```bash
chown -R 1000:1000 app-storage logs
```

Follow further instructions on [Self-Hosting Docs](https://docs.solidtime.io//self-hosting/guides/docker)

## Database access on the host machine

If you want to be able to access the database from the host machine, you need to expose the database port in the `docker-compose.yml` file.

> [!CAUTION]
> Only do this if you are not exposing this port to the internet. You can block this for example via a security group in most cloud providers or via a firewall on your local machine.

Uncomment the following lines in the `docker-compose.yml` file:

```yaml
  database:
    # ... other configuration
    ports:
      - '${FORWARD_DB_PORT:-5432}:5432'
    # ... other configuration
```

Set the `FORWARD_DB_PORT` environment variable to the port you want to use on the host machine

```dotenv
FORWARD_DB_PORT=5432
```
