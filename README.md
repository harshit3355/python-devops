# Counter Service — Containerised Flask App with CI to Amazon ECR

A deliberately small Flask service that counts POST requests and persists the count to a mounted volume. The application is the excuse; the point of the repository is the **delivery pipeline around it** — Docker build, resource-capped Compose deployment, SonarQube analysis, and a GitHub Actions workflow that publishes the image to Amazon ECR.

## Why this exists

Most "deploy a container" walkthroughs stop at `docker run`. This one covers the parts that actually matter once something is running for real:

- **State survives restarts** — the counter lives on a volume, not in process memory
- **The container is capped** — 256 MB and 0.5 CPU, so a runaway process cannot take the host down with it
- **The image is versioned and pushed to a private registry** rather than built on the server
- **Code quality is measured** on every change through SonarQube

It is a reference for the shortest complete path from source to a running, resource-bounded, registry-hosted container.

## API

| Method | Path | Behaviour |
| --- | --- | --- |
| `GET` | `/` | Returns the current count |
| `POST` | `/` | Increments the count and returns the new value |

The count is stored at `/data/counter.txt` inside the container, backed by the `./data` volume mount.

## Run it locally

```bash
pip install -r requirements.txt
python counter-service.py
```

```bash
curl localhost:8080          # -> 0
curl -X POST localhost:8080  # -> 1
curl -X POST localhost:8080  # -> 2
curl localhost:8080          # -> 2
```

The app writes to `/data`, so create that directory (or edit `COUNTER_FILE` in `counter-service.py`) before running outside a container.

## Run it with Docker Compose

```bash
docker compose up -d
curl -X POST localhost:80
```

The service is published on host port `80` mapped to container `8080`, with `./data` mounted at `/data` so the counter survives `docker compose down && docker compose up`.

To run your own build instead of the published ECR image, replace the `image:` line in `docker-compose.yaml` with `build: .`.

## Repository layout

| File | Purpose |
| --- | --- |
| `counter-service.py` | The Flask application |
| `requirements.txt` | Flask and gunicorn |
| `Dockerfile` | Container image definition |
| `.dockerignore` | Keeps the build context small |
| `docker-compose.yaml` | Deployment with volume, port mapping, and CPU/memory limits |
| `sonar-project.properties` | SonarQube scanner configuration |
| `.github/workflows/blank.yml` | CI: build and push to Amazon ECR |
| `README.Docker.md` | Docker scaffolding notes |

## CI/CD

The GitHub Actions workflow builds the image and pushes it to Amazon ECR. It needs these repository secrets:

| Secret | Purpose |
| --- | --- |
| `AWS_ACCESS_KEY_ID` | IAM credentials with ECR push permission |
| `AWS_SECRET_ACCESS_KEY` | Matching secret key |

The registry URI and region are set in `docker-compose.yaml` and in the workflow. Change both to point at your own AWS account.

## Notes and limits

- `docker-compose.yaml` pins format version `2.4` on purpose: it is the last version where `mem_limit` and `cpus` take effect without Swarm. Under Compose v3 you would need `deploy.resources`, which is ignored outside Swarm mode.
- The counter is a single file written by a single process. That is fine for one replica and **not** safe to scale horizontally — concurrent writers will lose increments. Move the counter to Redis or a database before adding replicas.
- There is no authentication. Anything that can reach the port can increment the counter. Put it behind a reverse proxy or a security group if it is exposed.
