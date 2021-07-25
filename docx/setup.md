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
      -p 8000:8000 \
      -p 8443:8443 \
      -p 10.12.0.10:8001:8001 \
      -p 127.0.0.1:8444:8444 \
      kong:2.5.0
      ```
- Bước 6: Kiểm tra  
  - Sử dụng câu lệnh:

   ```curl -i http://localhost:8001/```
```
Cách 2: Sử dụng package
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

curl -i http://localhost:8001/
```

##Cài đặt Konga
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