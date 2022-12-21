# Cài đặt KONG API
- Mục tiêu: Cài đặt Kong API Gateway sử dụng Postgres database

## Cách 1: Sử dụng docker
- Bước 1: Cài đặt docker trên server
- Bước 2: Tạo docker network
  - Sử dụng câu lệnh:
    ```docker network create kong-net```
- Bước 3: Chạy database image
  - Sử dụng câu lệnh:
      ```
      docker run -d --name kong-database \
      --network=kong-net \
      -p 5432:5432 \
      -e "POSTGRES_USER=kong" \
      -e "POSTGRES_DB=kong" \
      -e "POSTGRES_PASSWORD=kong" \
      postgres:9.6
      ```
- Bước 4: Config database:
  - Sử dụng câu lệnh:

      ```docker run --rm \
      --network=kong-net \
      -e "KONG_DATABASE=postgres" \
      -e "KONG_PG_HOST=kong-database" \
      -e "KONG_PG_USER=kong" \
      -e "KONG_PG_PASSWORD=kong" \
      -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
      kong:2.5.0 kong migrations bootstrap
      ```
- Bước 5: Chạy Kong
  - Sử dụng câu lệnh:
```
      docker run -d --name kong \
      --network=kong-net \
      -e "KONG_DATABASE=postgres" \
      -e "KONG_PG_HOST=kong-database" \
      -e "KONG_PG_USER=kong" \
      -e "KONG_PG_PASSWORD=kong" \
      -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
      -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
      -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
      -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
      -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
      -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
      -p 80:8000 \
      -p 443:8443 \
      -p 10.12.0.10:8001:8001 \
      -p 127.0.0.1:8444:8444 \
      kong:2.5.0
  ```
- Bước 6: Kiểm tra  
  - Sử dụng câu lệnh:
    ```curl -i http://localhost:8001/```

*Cách 2: Sử dụng package*
Bước 1: Cài đặt kong
Sử dụng câu lệnh:

curl -Lo kong.2.4.1.amd64.deb "https://download.konghq.com/gateway-2.x-ubuntu-$(lsb_release -cs)/pool/all/k/kong/kong_2.4.1_amd64.deb"
sudo dpkg -i kong.2.4.1.amd64.deb
Bước 2: Config database:
Vào postgres cli và thực hiện câu lệnh

CREATE USER kong; CREATE DATABASE kong OWNER kong;
Sau đó thực hiện câu lệnh:

kong migrations bootstrap [-c /path/to/kong.conf]
Bước 3: Chạy Kong
Sử dụng câu lệnh:

kong start [-c /path/to/kong.conf]
Bước 5: Kiểm tra
Sử dụng câu lệnh:

- ```curl -i http://localhost:8001/```


## Cài đặt Konga
- Sử dụng docker cài đặt Konga

- Bước 1: Cài đặt network:
  - Sử dụng câu lệnh:

    ```docker network create -d bridge main-link```
- Bước 2: Chạy docker
        ```
        
        docker run --rm -e POSTGRES_USER=konga -e POSTGRES_DB=konga -e POSTGRES_PASSWORD=konga -v data:/var/lib/postgresql/data --name postgres -d --network main-link postgres:9.6-alpine
        
        docker run --network main-link --rm pantsel/konga:latest -c prepare -a postgres -u postgresql://konga:konga@postgres:5432/konga
        
        docker run -p 1337:1337 \
         --network main-link \
         -e "TOKEN_SECRET=ffffssf" \
         -e "DB_ADAPTER=postgres" \
         -e "DB_HOST=postgres" \
         -e "DB_PORT=5432" \
         -e "DB_USER=konga" \
         -e "DB_PASSWORD=konga" \
         -e "DB_DATABASE=konga" \
         -e "NODE_ENV=production" \
         -d --name konga \
        pantsel/konga
        ```
Phần cài đặt Konga sẽ tiếp tục được cập nhật

# Cài đặt với docker-compose
```
version: "3"

networks:
 kong-net:
  driver: bridge

services:

  #######################################
  # Postgres: The database used by Kong
  #######################################
  kong-database:
    image: postgres:9.6
    restart: always
    networks:
      - kong-net
    environment:
      POSTGRES_USER: kong
      POSTGRES_DB: kong
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "kong"]
      interval: 5s
      timeout: 5s
      retries: 5

  #######################################
  # Kong database migration
  #######################################
  kong-migration:
    image: kong:latest
    command: "kong migrations bootstrap"
    networks:
      - kong-net
    restart: on-failure
    environment:
      KONG_PG_HOST: kong-database
    links:
      - kong-database
    depends_on:
      - kong-database

  #######################################
  # Kong: The API Gateway
  #######################################
  kong:
    image: kong:latest
    restart: always
    networks:
      - kong-net
    environment:
      KONG_PG_HOST: kong-database
      KONG_DATABASE: postgres 
      KONG_PROXY_LISTEN: 0.0.0.0:8000
      KONG_PROXY_LISTEN_SSL: 0.0.0.0:8443
      KONG_ADMIN_LISTEN: 0.0.0.0:8001
    depends_on:
      - kong-migration
      - kong-database
    healthcheck:
      test: ["CMD", "curl", "-f", "http://kong:8001"]
      interval: 5s
      timeout: 2s
      retries: 15
    ports:
      - "8001:8001"
      - "8000:8000"

  #######################################
  # Konga database prepare
  #######################################
  konga-prepare:
    image: pantsel/konga:next
    command: "-c prepare -a postgres -u postgresql://kong@kong-database:5432/konga_db"
    networks:
      - kong-net
    restart: on-failure
    links:
      - kong-database
    depends_on:
      - kong-database

  #######################################
  # Konga: Kong GUI
  #######################################
  konga:
    image: pantsel/konga:next
    restart: always
    networks:
        - kong-net
    environment:
      DB_ADAPTER: postgres
      DB_HOST: kong-database
      DB_USER: kong
      TOKEN_SECRET: km1GUr4RkcQD7DewhJPNXrCuZwcKmqjb
      DB_DATABASE: konga_db
      NODE_ENV: production
    depends_on:
      - kong-database
    ports:
      - "1337:1337"
```


# Hoặc tham khảo compose sau: https://github.com/tuannguyenssu/kong-examples/blob/master/kong-db-mode/kong-with-db-mode.yml
```
version: "3.7"

networks:
 kong:
  driver: bridge
  name: kong-network 

services:

  #######################################
  # Postgres: The database used by Kong
  #######################################
  kong-postgres-database:
    container_name: kong-postgres-database
    image: postgres:9.6
    restart: always
    networks:
      - kong
    environment:
      POSTGRES_USER: kong
      POSTGRES_DB: kong
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "kong"]
      interval: 60s
      timeout: 5s
      retries: 2

  #######################################
  # Kong database migration
  #######################################
  kong-migration:
    container_name: kong-migration
    image: kong
    command: "kong migrations bootstrap"
    networks:
      - kong
    restart: on-failure
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-postgres-database
    depends_on:
      - kong-postgres-database

  #######################################
  # Konga database prepare
  #######################################
  konga-prepare:
    container_name: konga-prepare
    image: pantsel/konga
    command: "-c prepare -a postgres -u postgresql://kong@kong-postgres-database:5432/konga_db"
    networks:
      - kong
    restart: on-failure
    depends_on:
      - kong-postgres-database

  #######################################
  # Konga: Kong GUI
  #######################################
  konga:
    container_name: konga
    image: pantsel/konga
    restart: always
    networks:
        - kong
    environment:
      DB_ADAPTER: postgres
      DB_HOST: kong-postgres-database
      DB_USER: kong
      NO_AUTH: "true"      
      DB_DATABASE: konga_db
      NODE_ENV: production
    depends_on:
      - konga-prepare 
      - kong-migration
      - kong-postgres-database          
    volumes:
    - ./konga/data:/app/kongadata      
    ports:
      - "1337:1337"    

  #######################################
  # Kong: The API Gateway
  #######################################
  kong:
    container_name: kong
    image: kong
    restart: unless-stopped
    networks:
      - kong
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-postgres-database
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_LOG_LEVEL: debug
      KONG_ADMIN_LISTEN: 0.0.0.0:8001, 0.0.0.0:8444 ssl
      KONG_PROXY_LISTEN: 0.0.0.0:8000, 0.0.0.0:8443 ssl, 0.0.0.0:9080 http2, 0.0.0.0:9081 http2 ssl
    depends_on:
      - kong-migration
      - kong-postgres-database
    healthcheck:
      test: ["CMD", "kong", "health"]
      interval: 120s
      timeout: 10s
      retries: 3
    volumes:
    - ./compose/kong/logs:/usr/local/kong/logs       
    - ./compose/kong/logs/log.txt:/usr/local/kong/logs/log.txt       
    - ./compose/kong/config/nginx.conf:/usr/local/kong/nginx.conf       
    - ./compose/kong/config/nginx-kong.conf:/usr/local/kong/nginx-kong.conf       
    ports:
      - "8001:8001"
      - "8444:8444"
      - "8000:8000"
      - "8443:8443"
      - "9080:9080"
```
