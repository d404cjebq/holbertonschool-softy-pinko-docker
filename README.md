# Softy Pinko Docker

A step-by-step Docker project that builds the infrastructure for a small web application: a **reverse proxy / load balancer**, **two API servers**, and one **front-end static-content server**, all orchestrated with Docker Compose.

## Architecture

```
                    +-----------------------------+
   Browser  ----->  |  proxy (Nginx, port 80)     |
                    |  reverse proxy + load balancer
                    +--------------+--------------+
                                   |
              location /           |           location /api
          +------------------------+------------------------+
          |                                                 |
+---------v----------+                        +-------------v-------------+
| front-end (Nginx)  |                        | back-end (Flask, port 5252)|
| static files :9000 |                        |  scaled to 2+ containers   |
+--------------------+                        |  Round Robin balancing     |
                                              +----------------------------+
```

- The proxy is the **only entry point** (host port 80).
- Requests to `/` go to the front-end static server.
- Requests to `/api` are load-balanced between the API servers using **Round Robin**.
- The client never talks to the front-end or the back-end directly.

## Requirements

- [Docker Desktop](https://www.docker.com/) installed **and running** (otherwise you get `Cannot connect to the Docker daemon`)
- Docker Compose (`docker-compose` or `docker compose`)

## Tasks

| Task | Directory | Description |
|------|-----------|-------------|
| 0 | `task0` | First Docker image: Ubuntu, `apt-get update` and `upgrade`, prints `Hello, World!` |
| 1 | `task1` | Back-end: install Python3, pip3 and Flask; Flask API on port 5252 with `/api/hello` |
| 2 | `task2` | Front-end: Nginx static server (port 9000) serving `softy-pinko-front-end`; project split into `back-end/` and `front-end/` |
| 3 | `task3` | Connect front-end and back-end: dynamic `<h1>` filled by jQuery AJAX; CORS enabled in Flask with `flask-cors` |
| 4 | `task4` | Docker Compose: run both services with a single `docker-compose up` |
| 5 | `task5` | Nginx proxy server (port 80) routing `/` to the front-end and `/api` to the back-end; front-end and back-end ports are no longer exposed to the host |
| 6 | `task6` | Horizontal scaling: run 2 API servers with Round Robin load balancing |

## Quick start (final version, task6)

```bash
cd task6
docker-compose build
docker-compose up --scale back-end=2
```

Then open <http://localhost>. The text `Hello, World!` appears above the main title, fetched from the API through the proxy.

Reload the page several times and watch the terminal: requests alternate between `task6-back-end-1` and `task6-back-end-2`.

Stop and clean up with:

```bash
docker-compose down
```

## Running a single task

Task 0:

```bash
cd task0
docker build -f ./Dockerfile -t softy-pinko:task0 .
docker run -it --rm --name softy-pinko-task0 softy-pinko:task0
```

Task 1:

```bash
cd task1
docker build -f ./Dockerfile -t softy-pinko:task1 .
docker run -p 5252:5252 -it --rm --name softy-pinko-task1 softy-pinko:task1
# http://localhost:5252/api/hello
```

Task 2 (front-end only):

```bash
cd task2
docker build -f ./front-end/Dockerfile -t softy-pinko-front-end:task2 ./front-end
docker run -p 9000:9000 -it --rm --name softy-pinko-front-end-task2 softy-pinko-front-end:task2
# http://localhost:9000
```

Task 3 (two terminals):

```bash
cd task3
# Terminal 1
docker build -f ./back-end/Dockerfile -t softy-pinko-back-end:task3 ./back-end
docker run -p 5252:5252 -it --rm --name softy-pinko-back-end-task3 softy-pinko-back-end:task3
# Terminal 2
docker build -f ./front-end/Dockerfile -t softy-pinko-front-end:task3 ./front-end
docker run -p 9000:9000 -it --rm --name softy-pinko-front-end-task3 softy-pinko-front-end:task3
```

Tasks 4 to 6:

```bash
cd task4   # or task5 / task6
docker-compose build
docker-compose up
```

## Project structure (task6)

```
task6/
├── 2-api-servers.txt          # docker-compose command used to run 2 API servers
├── docker-compose.yml
├── back-end/
│   ├── Dockerfile             # Ubuntu + python3 + pip3 + flask + flask-cors
│   └── api.py                 # Flask app, /api/hello on port 5252
├── front-end/
│   ├── Dockerfile             # nginx + static site
│   ├── softy-pinko-front-end.conf
│   └── softy-pinko-front-end/ # cloned static website (index.html, assets)
└── proxy/
    ├── Dockerfile             # nginx
    └── proxy.conf             # routes / and /api, listens on port 80
```

## Key concepts

- **Dockerfile**: instructions to build an image (`FROM`, `RUN`, `COPY`, `WORKDIR`, `CMD`, `EXPOSE`).
- **Image vs container**: an image is the template, a container is a running instance of it.
- **Port mapping**: `HOSTPORT:CONTAINERPORT`, for example `-p 5252:5252`. Only mapped ports are reachable from your machine.
- **Reverse proxy**: a server that receives client requests and forwards them to internal servers, hiding them from the client.
- **Load balancer / Round Robin**: requests are distributed to each server in turn (A, B, A, B, ...).
- **Docker Compose**: one `docker-compose.yml` describes all services, their builds, ports and dependencies.
- **Service names as hostnames**: inside a Compose network, Docker DNS resolves `front-end` and `back-end` to the containers' internal IPs, which is why `proxy.conf` uses `http://front-end:9000` and `http://back-end:5252`.
- **`expose` vs `ports`**: `expose` opens a port only to other containers; `ports` publishes it on the host. Using `expose` for the back-end is what allows scaling it to several containers without port conflicts.
- **CORS**: needed in tasks 3 and 4 because the browser called the API on a different origin (port 5252). With the proxy (task 5), the call is same-origin (`/api/hello`).
- **Scaling**: `docker-compose up --scale back-end=2` starts two instances of the `back-end` service.

## Common problems

| Problem | Fix |
|---------|-----|
| `Cannot connect to the Docker daemon` | Open Docker Desktop and wait until it is running |
| `port is already allocated` | Stop old containers: `docker ps`, then `docker stop <name>` or `docker-compose down` |
| Old page still shows after editing `index.html` | Rebuild the image (`docker-compose build`) and hard refresh the browser |
| `git add task1` fails with `did not match any files` | You are inside `task1`; run `git add .` or go to the repo root first |
| Page loads but no `Hello, World!` | Check the browser console and make sure the back-end container is running |
| 403 / 404 from the front-end | Check that `index.html` is directly inside `softy-pinko-front-end/` and matches `root` in the Nginx conf |

## Author

Holberton School project: `holbertonschool-softy-pinko-docker`
