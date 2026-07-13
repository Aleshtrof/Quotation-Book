Проблемы при сборке
1. docker compose ps
WARN[0000] The "JAVA_OPTS" variable is not set. Defaulting to a blank string.
- выполнена корректирока переменной
было:
JAVA_OPTS: "${JAVA_OPTS} -Djava.io.tmpdir=/tmp"
стало:
JAVA_OPTS=-Djava.io.tmpdir=/tmp

2. Сборка QB.Java.Obs.Dockerfile
- пришлось заменить 
FROM ${DOCKER_REGISTRY}/${MAVEN_BASE_IMAGE} AS builder
на
FROM maven:3.9.6-eclipse-temurin-11 AS builder
т.к. образ исходный образ не доступен по ссылке quay.io/dev_zone/redos/ubi8/maven-396
- по причине использования другого образа закомментированы строки
#COPY ./docker/certs /etc/pki/ca-trust/source/anchors
#update-ca-trust 

3. Некорректный endpoint в healthcheck в QB.Dotnet.Obs.Dockerfile
- Корректный healthcheck выполнен через docker compose (вместо /health -> /status)

4. Сервис quotation-book-postgres
В .env отсутствовала стандартная переменная POSTGRES_PASSWORD
- Переменная добавлена в docker compose в виде hardcode

5. Сервис quotation-book-keycloak
В .env отсутствовала стандартная переменная KC_HOSTNAME
- Добавлены переменные в .env и compose
KC_HOSTNAME=localhost:8443/keycloak
KC_HOSTNAME_STRICT=false
KC_HOSTNAME_STRICT_HTTPS=true
KC_PROXY_HEADERS=xforwarded
- изменена переменна KC_HTTP_ENABLED с false на true

6. сервис quotation-book-pgadmin
В .env отсутствовали стандартные переменные PGADMIN_DEFAULT_EMAIL и PGADMIN_DEFAULT_PASSWORD
- Переменная добавлена в docker compose:
PGADMIN_DEFAULT_EMAIL: "${PG_ADMIN_USER}"
PGADMIN_DEFAULT_PASSWORD: "${PG_ADMIN_PASSWORD}"

7. Сервис quotation-book-rabbitmq
В .env отсутствовали переменные, которые используются в образе и влияют на подключение quotation-book-backend-worker
- Переменные добавлены в docker compose:
      ENABLE_SSL: "${ENABLE_SSL_RABBITMQ}"
      SSL_PORT: "${RABBITMQ_PORT_SSL}"
      CA_CERT_FILE: "${CA_CERTIFICATE}"
      CERT_FILE: "${RABBITMQ_CERT_FILE}"
      KEY_FILE: "${RABBITMQ_KEY_FILE}"

8.Сервис quotation-book-frontend
- Внесено изменение в конфиг nginx default-location-conf.template
1. было
location /keycloak/ {
    proxy_pass https://${KEYCLOAK_HOST}:${KEYCLOAK_PORT}/;
стало
location /keycloak/ {
    proxy_pass http://${KEYCLOAK_HOST}:${KEYCLOAK_PORT}/;
- Изменена переменная в .env KEYCLOAK_PORT c 1443 на 8081

2. скорректирован alias
было
alias /opt/app-root/static/
стало
alias /opt/app-root/src/static/


9.Сервис quotation-book-backend
- Внесено изменение в маршруты для аутентификации через Keycloak auth.py, изменён url с https на http для:
token_url
userinfo_url
token_refresh_url

10. Сервис quotation-book-backend-worker
Переопределена переменная       REDIS_PORT: "${REDIS_PORT_TLS}" (6379 -> 6380)
