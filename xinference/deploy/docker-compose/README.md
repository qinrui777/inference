
# Docker Compose deployment guide

This directory contains a Compose-based deployment for Xinference. It is intended for local development and small-team environments, with optional helper services that can be enabled on demand.

## Quick start

1. Copy the example environment file to a local `.env` file:

   ```sh
   cp example.env .env
   ```

2. Review and adjust the values in `.env` for your requirements.

3. Start the default stack:

   ```sh
   docker compose up -d
   ```

4. Open the service at:

   - Xinference URL: http://localhost:9997

## Environment variables

The Compose file uses the following variables from `.env`:

- `VERSION`: Xinference image tag.
- `XINFERENCE_MODEL_SRC`: model source used by Xinference.
- `HOST_XINFERENCE_BASE`: host home directory used for persistent model/cache data.

A ready-to-copy example is provided in `example.env`.

## Optional services

### Enable the PyPI mirror

```sh
COMPOSE_PROFILES=pypiserver docker compose up -d
```

This starts the optional helper service:

- PyPI mirror: http://localhost:8080

## Operational commands

Check running containers:

```sh
docker compose ps
```

View logs:

```sh
docker compose logs -f xinference
```

Stop and remove the stack:

```sh
docker compose down
```

## Notes for team use

- Keep `.env` local and do not commit your machine-specific values.
- Use `example.env` as the shared template for onboarding new developers.
- Optional services are profile-based so the default startup stays lightweight.