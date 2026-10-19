# Example: Docker + Nginx + Let's Encrypt

This example shows how to run solidtime with Docker, Nginx, and PostgreSQL. The stack starts on HTTP so Let's Encrypt can verify the domain, then switches to HTTPS once the first certificate is issued.

## Prerequisites

- [Docker](https://docs.docker.com/engine/install/)
- Public DNS record pointing your domain to the host running this compose stack

## Installation

1. Copy the example files and adjust the environment variables to your needs

```bash
cp laravel.env.example laravel.env
cp .env.example .env
```
Refer to the [Self-Hosting Docker guide](https://docs.solidtime.io/self-hosting/guides/docker#1-choose-the-example-that-fits-your-needs-the-best) for details on configuring these files and the broader deployment flow.

2. (Only with Linux/Docker Engine) Correct the permissions of the `app-storage` and `logs` directories.

```bash
chown -R 1000:1000 app-storage logs
```

3. Start the services.

```bash
docker compose up -d
```

4. Request the first Let's Encrypt certificate. Replace the domain and email with your own values.

```bash
docker compose run --rm certbot certonly \
    --webroot -w /var/www/html \
    --domain solidtime.example.com \
    --email you@example.com \
    --agree-tos --non-interactive
```

5. Copy the SSL template, update the domain placeholders in `nginx/ssl.conf`, then restart Nginx.

```bash
cp nginx/ssl.conf.template nginx/ssl.conf
docker compose restart nginx
```

6. Start certbot so renewals run automatically every 12 hours.

```bash
docker compose up -d certbot
```

To enforce HTTPS after verifying the certificate, update the `/` location in `nginx/nginx.conf` to `return 301 https://$host$request_uri;`.

