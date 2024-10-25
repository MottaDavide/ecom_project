## Docker
In the modern software development landscape, Docker has emerged as a game-changer. But why is it so revered? Let’s embark on this journey to understand Docker’s significance and how it integrates seamlessly with our e-commerce platform.

### Why Docker?
Imagine a world where you can wrap your application, with all its dependencies, into a single package and then run it consistently across various environments. That’s Docker for you! It ensures that “it works on my machine” is a phrase of the past. Docker containers encapsulate everything an application needs to run, ensuring consistency, scalability, and isolation.

### Create a Docker file
The heart of Docker is the Dockerfile. It’s a script that contains all the commands a user could call to assemble an image. For our `order_service`, the Dockerfile looks like this:
```
FROM custom-python-flyway:latest 
WORKDIR /app 
COPY . /app 
RUN pip install --upgrade pip \
 && pip install poetry \
 && poetry config virtualenvs.create false \
 && poetry install --no-dev 
EXPOSE 80 
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "80"]
```

This Dockerfile does a few things:

- Uses a custom Python image with Flyway included.
- Sets the working directory in the container to /app.
- Copies the current directory content into the container.
- Installs necessary Python packages and dependencies.
- Exposes port 80 for the application.
- Specifies the command to run when the container starts.

While Docker Hub offers a plethora of pre-built images for almost every use case, sometimes, the need arises to craft a custom image tailored to specific requirements. For our e-commerce platform, integrating database migrations into the service startup was one such requirement. Enter Flyway — a database migration tool.

### Custom Image
When services start, it’s crucial that the database schema is in the expected state. Flyway ensures this by applying version-controlled scripts to the database, bringing it to the desired state. But how do we ensure Flyway runs before our service starts? The answer is a custom Docker image that has both Python (for our service) and Flyway (for migrations).

In the `ecom_project` directory, we have a Dockerfile dedicated to this task:

```
FROM python:3.9-slim 

RUN apt-get update \ 
 && apt-get install -y wget tar \ 
 && pip install --upgrade pip

RUN wget -O flyway.tar.gz https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/7.15.0/flyway-commandline-7.15.0-linux-x64.tar.gz \ 
 && tar -xzf flyway.tar.gz -C /opt \ 
 && ln -s /opt/flyway-7.15.0/flyway /usr/local/bin
 ```

Here’s a breakdown of what’s happening:

- We start with the `python:3.9-slim` image, a lightweight Python image.
- We update the package list and install `wget` (to download files) and `tar` (to extract archives).
- We download the Flyway command-line tool and extract it to `/opt`.
- A symbolic link is created to ensure the `flyway` command is available globally in the container.

With this custom image, every time a service container starts, it has both Python and Flyway at its disposal, ensuring smooth service initialization and database migrations.

### Orchestrating Multi-container Applications

While Docker is fantastic for containerizing a single application, in the real world, applications often interact with other services. Enter `docker-compose`. It's a tool to define and run multi-container Docker applications. For our e-commerce platform, the `docker-compose.yml` in the `ecom_project` directory defines the orchestration:

```
version: '3'
services:
  zookeeper:
    image: wurstmeister/zookeeper:3.4.6
    ports:
      - "2181:2181"
  kafka:
    image: wurstmeister/kafka:latest
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_LISTENERS: INSIDE://kafka:9093,OUTSIDE://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INSIDE:PLAINTEXT,OUTSIDE:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INSIDE
      KAFKA_LISTENERS: INSIDE://0.0.0.0:9093,OUTSIDE://0.0.0.0:9092
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    depends_on:
      - zookeeper
  order_postgres:
    image: postgres:latest
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: orderdb
    ports:
      - "5433:5432"
    volumes:
        - order_postgres_data:/var/lib/postgresql/data
  order_service:
    build: ./services/order_service
    environment:
      DATABASE_URL: postgresql://user:password@order_postgres:5432/orderdb
    ports:
      - "80:80"
    depends_on:
      - kafka
      - order_postgres

volumes:
  order_postgres_data:
```

This file describes:

- How services like Kafka, Zookeeper, and PostgreSQL are set up.
- The dependencies between these services.
- The environment variables and volumes required for persistence and configuration.

## API
In the modern world of web development, crafting a RESTful API is more of a necessity than a luxury. For our e-commerce platform, we needed a robust, fast, and intuitive framework. Enter FastAPI — a modern, fast (high-performance) web framework for building APIs with Python based on standard Python type hints.

### FastAPI
FastAPI is not just another framework; it’s a game-changer. Here’s why:

- Type Hints: With Python’s type hints, FastAPI allows for intuitive function signatures, ensuring that the data you receive and send is of the expected type.
- Automatic API Docs: Forget manually updating Swagger or ReDoc. FastAPI generates it for you.
- Performance: Comparable to NodeJS and Go, it’s one of the fastest frameworks for Python, especially with I/O bound operations.

## SQLAlchemy
While FastAPI handles the web part, we needed an ORM (Object Relational Mapper) to interact with our database. SQLAlchemy, a popular Python library, was our choice. It provides a set of high-level API to connect to relational databases. Using SQLAlchemy, we can model our tables as classes and query them without writing raw SQL.