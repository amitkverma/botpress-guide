# Deploying with Docker

These steps shows how to deploy botpress using Docker

## Single container deployment

Download botpress file from source and run it.

```
docker run -d \
--name botpress \
-p 3000:3000 \
-v botpress_data:/botpress/data \
botpress/server:$TAG
```
This will get you up and running locally on port 3000. Try visiting visiting http://localhost:3000/

## Deployment using Docker compose
create a new `botpress-docker-compose.yml` file in your project directory and copy-paste the following content.

```
version: '3'

services:
  botpress:
    image: botpress/server
    command: /botpress/bp
    expose:
      - 3000
    environment:
      - DATABASE_URL=postgres://postgres:secretpw@postgres:5435/botpress_db
      - REDIS_URL=redis://redis:6379?password=redisPassword
      - BP_MODULE_NLU_DUCKLINGURL=http://botpress_lang:8000
      - BP_MODULE_NLU_LANGUAGESOURCES=[{"endpoint":"http://botpress_lang:3100"}]
      - CLUSTER_ENABLED=true
      - BP_PRODUCTION=true
      - BPFS_STORAGE=database
    depends_on:
      - botpress_lang
      - postgres
      - redis
    volumes:
      - ./botpress/data:/botpress/data
    ports:
      - "3000:3000"

  botpress_lang:
    image: botpress-lang
    command: bash -c "./duckling -p 8000 & ./bp lang --langDir /botpress/lang --port 3100"
    expose:
      - 3100
      - 8000
    volumes:
      - ./botpress/language:/botpress/lang

  postgres:
    image: postgres:11.2-alpine
    expose:
      - 5435
    environment:
      PGPORT: 5435
      POSTGRES_DB: botpress_db
      POSTGRES_PASSWORD: secretpw
      POSTGRES_USER: postgres
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:5.0.5-alpine
    expose:
      - 6379
    command: redis-server --requirepass redisPassword
    volumes:
      - redisdata:/data

volumes:
  pgdata:
  redisdata:
```

To run the docker compose file using following command
```
docker-compose -f botpress-docker-compose.yml up
```
This will get you up and running locally on port 3000. Try visiting visiting http://localhost:3000/
