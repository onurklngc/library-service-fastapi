# Library Service

A REST API for running a small library: it manages books and patrons, handles checkouts and returns, and reports
overdue loans. Scheduled background jobs send daily overdue reminders and write a weekly inventory report.

Built with FastAPI, SQLAlchemy 2 and MySQL, with Celery workers and Celery Beat on Redis. Everything runs in Docker,
and the test suite runs on every push via GitHub Actions.

## Architecture

```
            ┌──────────────┐        ┌─────────┐
client ───► │ FastAPI app  │ ─────► │  MySQL  │ ◄────────────┐
            └──────────────┘        └─────────┘              │
                                                             │
            ┌──────────────┐        ┌─────────┐       ┌──────┴───────┐
            │ Celery Beat  │ ─────► │  Redis  │ ────► │ Celery worker│
            │ (scheduler)  │        │ (broker)│       │ (jobs)       │
            └──────────────┘        └─────────┘       └──────────────┘
```

- **API** (`main.py`): request validation with Pydantic (ISBN-10/13 and email checks), persistence via SQLAlchemy.
- **Scheduler** (`celery_app.py`): cron-style schedule; hours are configurable through environment variables.
- **Jobs** (`src/tasks.py`): `send_overdue_reminders` (daily) and `generate_weekly_reports` (weekly).

## API

| Method | Path | Description |
|---|---|---|
| `POST` | `/books/` | Add a book |
| `GET` | `/books` | List books |
| `GET` `PUT` `DELETE` | `/books/{book_id}` | Read, update or delete a book |
| `POST` | `/patrons/` | Register a patron |
| `GET` | `/patrons` | List patrons |
| `GET` `PUT` `DELETE` | `/patrons/{patron_id}` | Read, update or delete a patron |
| `POST` | `/checkout/{book_id}/patron/{patron_id}` | Check out a book (`checkout_period` in days, default 14) |
| `POST` | `/return/{book_id}` | Return a book |
| `GET` | `/books/checkedout/` | List checked-out books |
| `GET` | `/books/patron/{patron_id}` | List books held by a patron |
| `GET` | `/books/overdue/` | List overdue books |

Interactive documentation is served at `/docs` (Swagger UI).

## Environment Setup

## a) Build with docker-compose

1. **Create Environment Variables:**
    - Create your own `.env` file by using the [sample](envs/.env_sample_for_docker_compose) provided:
      ```sh
      cp envs/.env_sample_for_docker_compose .env
      ```

2. **Build and Start Containers:**
    - In the project directory, run the following command to build and start the services:
      ```sh
      docker-compose up --build
      ```

## b) Build manually

Alternatively, you can start each container separately.

1. **Create Environment Variables:**
    - Use the [sample file](envs/.env_sample_for_manual_builds) for manual builds:
      ```sh
      cp envs/.env_sample_for_manual_builds .env
      set -o allexport
      source .env
      ```

2. **Set Up MySQL Server:**
    - Start the MySQL Docker container:
      ```sh
      docker run -p ${MYSQL_PORT}:3306 --detach --name=$MYSQL_CONTAINER_NAME --env-file=.env mysql
      ```
    - Wait for the MySQL server to be ready (approximately 10 seconds).
    - Create the database schema:
      ```sh
      docker exec -i $MYSQL_CONTAINER_NAME mysql -uroot -p$MYSQL_ROOT_PASSWORD -e "CREATE DATABASE library CHARACTER SET utf8;"
      ```

3. **Set Up Redis:**

    - Start Redis:
      ```sh
      docker run --rm --detach -p ${REDIS_PORT}:6379 --name $REDIS_CONTAINER_NAME redis:latest
      ```
4. **Set Up FastAPI Server:**

    - Build and run the FastAPI backend server:
      ```sh
      docker build -t onurklngc/library-backend:0.0.1 .
      docker stop $LIBRARY_BACKEND_CONTAINER_NAME && docker rm $LIBRARY_BACKEND_CONTAINER_NAME
      docker run --env-file=.env -p ${LIBRARY_BACKEND_PORT}:8081 --name $LIBRARY_BACKEND_CONTAINER_NAME \
      onurklngc/library-backend:0.0.1
      ```

5. **Set Up Celery Applications:**

    - Start Celery Worker:
      ```sh
      docker run --env-file=.env --name $CELERY_CONTAINER_NAME --entrypoint "" onurklngc/library-backend:0.0.1 celery -A celery_app worker -l info
      ```
    - Start Celery Beat:
      ```sh
      docker run --env-file=.env --name $CELERY_BEAT_CONTAINER_NAME --entrypoint "" onurklngc/library-backend:0.0.1 celery -A celery_app beat -l info
      ```

## Accessing the API

Once the FastAPI server is running, you can interact with the API using Swagger UI.
Visit [http://localhost:8081/docs](http://localhost:8081/docs) in your browser.

## Checking Celery Jobs

Celery is used in this project for handling asynchronous tasks, such as sending daily reminders for overdue books and
generating weekly reports. These tasks are managed by Celery Worker and scheduled by Celery Beat.

### Monitoring Celery Jobs

- **Daily Overdue Book Reminders:**
    - Celery sends reminders to patrons for overdue books every day.
    - You can monitor the execution of this task by checking the logs of the Celery Worker container. The logs will show
      when reminders are sent and to whom.

- **Weekly Reports:**
    - Every week, Celery generates a report of book checkouts and other relevant statistics.
    - This report generation is also logged, and you can check the output in the Celery Worker logs.

### Viewing Logs

To view the logs where the Celery jobs are printed, use the following command to view real-time logs from the Celery
Worker:
```sh
docker logs -f $CELERY_CONTAINER_NAME
```

These logs will show the execution of scheduled tasks, such as sending daily overdue book reminders and generating
weekly reports. If the tasks are running as expected, you will see log entries indicating their completion.

### Adjusting the Scheduler

If you want to see the job result logs sooner, you can adjust the cron job scheduler in the Celery Beat configuration:

- Modify the scheduling frequency of the tasks to a shorter interval for testing or quicker feedback.

## Testing

1. **Run Unit Tests:**
    - The tests use a local SQLite database, so no MySQL or Redis is needed. To run them:
      ```sh
      pytest
      ```

2. **Populate Database for Testing:**
    - You can populate the database with test data and validate the endpoints by running:
      ```sh
      python tests/populate.py
      ```

## Additional Information

- **Logs and Monitoring:**
    - Logs for Celery Worker and Celery Beat can be monitored in real-time using the `-l info` flag.

- **Scaling and Performance:**
    - The services can be scaled easily by adjusting the `docker-compose.yml` or by running multiple instances of the
      containers.
