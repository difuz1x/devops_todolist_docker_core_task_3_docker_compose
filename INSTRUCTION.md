# TodoApp + MySQL — Docker Compose runbook

A single command brings up the whole stack: a MySQL database and the Django
TodoApp, connected over a private bridge network, with todos stored on a
persistent volume.

## Prerequisites

- Docker Engine with the Compose plugin (`docker compose version` should print v2.x).
  If you only have the standalone binary, use `docker-compose` (with a hyphen)
  in place of `docker compose` below; the arguments are identical.
- In `todolist/settings.py`, the `DATABASES` host must point at the **compose
  service name**, not at a container IP. Compose runs an embedded DNS resolver,
  so `mysql` resolves to the database container automatically. The old "look up
  the container IP" step is no longer needed.

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.mysql",
        "NAME": "app_db",
        "USER": "app_user",
        "PASSWORD": "1234",
        "HOST": "mysql",   # <- service name from docker-compose.yml
        "PORT": "3306",
    }
}
```

## Project layout

```
.
├── docker-compose.yml     # defines the mysql + pythonapp services
├── Dockerfile             # builds the Django app image
└── Dockerfile.mysql       # builds the MySQL image
```

## Start the stack

```bash
docker compose up --build
```

This builds both images, starts the `mysql` container, then starts the
`pythonapp` container. The app's `ENTRYPOINT` runs the database migrations and
launches the dev server in one shot:

```
python manage.py migrate && python manage.py runserver 0.0.0.0:8080
```

Run it detached (in the background) instead:

```bash
docker compose up --build -d
```

Once it is up, the app is available at **http://localhost:8080**.

> First boot note: `depends_on` waits for the MySQL container to start, not for
> the server inside it to finish initializing. On a cold start the app may fail
> its first migration attempt and restart a few times. The `restart:
> unless-stopped` policy retries automatically, so it settles on its own after
> MySQL is ready.

## Check status and logs

```bash
docker compose ps              # container state and ports
docker compose logs -f         # follow logs from both services
docker compose logs -f pythonapp   # only the app
docker compose logs -f mysql       # only the database
```

## Stop the stack

```bash
docker compose stop            # stop containers, keep them for a fast restart
docker compose start           # resume the stopped containers
```

Tear it down completely:

```bash
docker compose down            # remove containers + network, KEEP the data volume
docker compose down -v         # also delete the MySQL-local volume (wipes all todos)
```

## Data persistence

Todos live in the named volume `MySQL-local`, mounted at `/var/lib/mysql`
inside the database container. The data survives `docker compose down` and any
container rebuild. Only `docker compose down -v` (or deleting the volume by
hand) destroys it.

Inspect it if needed:

```bash
docker volume ls               # MySQL-local should be listed
docker volume inspect <project>_MySQL-local
```

## Rebuild after code changes

```bash
docker compose up --build      # rebuild changed images and restart
```

## Connect to the database directly (optional)

```bash
docker compose exec mysql mysql -u app_user -p1234 app_db
```

## Troubleshooting

- **App can't reach the database / `Can't connect to MySQL server`**: confirm
  `HOST` in `settings.py` is `mysql` (the service name), not a hardcoded IP. A
  leftover IP from the manual `docker run` setup is the usual cause.
- **Port already allocated (`3306` or `8080`)**: a local MySQL or another
  container is using the port. Stop it, or change the host side of the mapping
  in `docker-compose.yml` (for example `"3307:3306"`) and reconnect on the new
  port.
- **Stale data after schema changes**: if migrations conflict with an old
  database state during testing, wipe the volume with `docker compose down -v`
  and bring the stack back up for a clean DB.

## (Optional) Publish the images to Docker Hub

The MySQL service is already named `difuz1x/mysql-local:1.0.0` in the compose
file. To push it, and to tag and push the app image:

```bash
docker compose build
docker tag <project>-pythonapp difuz1x/todoapp:2.0.0
docker push difuz1x/mysql-local:1.0.0
docker push difuz1x/todoapp:2.0.0
```

Published images:

- https://hub.docker.com/repository/docker/difuz1x/todoapp
- https://hub.docker.com/repository/docker/difuz1x/mysql-local