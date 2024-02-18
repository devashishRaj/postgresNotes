# Advance topics

We are going to create a docker compose file to move the demo database into the image and 
then open it
- go to terminal 
  - `vim .env`
    - paste the following and enter the valid paths:
    ```env
    DEMO_DATABASE_PATH="<path to demo file>/demo-small-en-20170815.sql"
    DATABASE_BACKUP="<path to db backup"
    #POSTGRES_DB="flight"
    POSTGRES_USER="flight"
    POSTGRES_PASSWORD="flight"
    ```
  
  - `vim docker-compose.yml`
    - paste the following
    ```env 
    version: '3'
    services:
    postgres:
    image: postgres:latest
    restart: always  
    container_name: postgresintro
    environment:
    POSTGRES_USER: ${POSTGRES_USER}
    POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}  
    ports:
    - "5432:5432"
    volumes:
      - ${DEMO_DATABASE_PATH}:/demo-small-en-20170815.sql
      - ${DATABASE_BACKUP}:/var/lib/postgresql/data
      ```
  
  - run : `docker composer up -d`
    -  `docker exec -it postgresintro psql -d flight -U flight`
    -  `\i demo-small-en-20170815.sql`
    - db will automatically change to demo

  - to change to database demo next time, 
    - `docker exec -it postgresintro psql -d demo -U flight`
    - OR : `\c demo` inside psql