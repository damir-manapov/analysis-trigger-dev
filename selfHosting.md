# Self Hosting

Options:
* Docker (by docker compose)
* Kubernetes

Components:
* UI
* Redis
* Postgres
* Electric (?)
* ClickHouse (Why do they need it?)
* Object storage (Optional)
* Registry (What is it? Optional)
* Worker supervisor
* Workers

## Compose

They use https://github.com/Tecnativa/docker-socket-proxy to secure access to docker socket to manage workers by supervisor.
