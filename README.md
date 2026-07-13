Выполненны изменения:
1. Изменена переменная в JAVA_OPTS .env

| было | стало |
|-------------------------------------------------|---------------------------------|
| JAVA_OPTS: "${JAVA_OPTS} -Djava.io.tmpdir=/tmp" | JAVA_OPTS=-Djava.io.tmpdir=/tmp |
Причина: Некорректный формат переменной


2. Заменён образ в dockerfile QB.Java.Obs.Dockerfile

| было | стало |
|--------------------------------------------------------|------------------------------------------------|
| FROM ${DOCKER_REGISTRY}/${MAVEN_BASE_IMAGE} AS builder | FROM maven:3.9.6-eclipse-temurin-11 AS builder |

2.1 Зкоментированы комнады в dockerfile QB.Java.Obs.Dockerfile
#COPY ./docker/certs /etc/pki/ca-trust/source/anchors
#update-ca-trust

Причина: Репозиторий с образом недоступен по адресу quay.io/dev_zone/redos/ubi8/maven-396

3. Healtcheck из образа QB.Dotnet.Obs.Dockerfile переписан через docker compose
Причина: Некорректный endpoint: вместо /health должен быть /status


3.1 Изменён код в файле Program.cs для сервиса quotation-book-dotnet-obs
            // --- ДОБАВЛЕНО: PostgreSQL проверяем через TCP ---
            if (service.Key == "PostgreSQL")
            {
                try
                {
                    using var tcp = new TcpClient();

                    var host = service.Value
                        .Replace("http://", "")
                        .Replace("https://", "")
                        .Split(':')[0];

                    await tcp.ConnectAsync(host, 5432);

                    statuses[service.Key] = true;
                }
                catch
                {
                    statuses[service.Key] = false;
                }

                continue;
            }

Причина: Исходный код выполнял подключение к postgres по http { "PostgreSQL", "http://quotation-book-postgres:5432" }

4. В .env добавлены стандартные переменные для сервиса quotation-book-postgres
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
Причина: Переменные отсутсовали в файле .env

4.1 В docker compose переопределены переменные для сервисов quotation-book-postgres*
      KEYCLOAK_DB_USER: "${KC_DB_USERNAME}"
      KEYCLOAK_DB_PASSWORD: "${KC_DB_PASSWORD}"
      KEYCLOAK_DB_NAME: "${KC_DB_NAME}"

Причина: Несоответствие переменных в docker-entrypoint.sh и файле .env

5. В .env добавлены стандартные переменные для сервиса quotation-book-keycloak
KC_HOSTNAME=localhost:8443/keycloak
KC_HOSTNAME_STRICT=false
KC_HOSTNAME_STRICT_HTTPS=true
KC_PROXY_HEADERS=xforwarded

Причина: Переменные необходимы для корректной работы keycloak за обратным прокси.
KC_HOSTNAME - определяет внешний адрес. доступный пользователю
KC_HOSTNAME_STRICT - разрешает принимать запросы, даже если значение заголовка Host не совпадает с KC_HOSTNAME (при работе через обратный прокси)
KC_HOSTNAME_STRICT_HTTPS - позволяет считать внешний адрес HTTPS и генерировать ссылки с протоколом https.
KC_PROXY_HEADERS - позволяет принимать заголовки от nginx

5.1 Изменено значение переменной KC_HTTP_ENABLED
| было | стало |
|-------|------|
| false | true |

Причина: Необходимо для принятия http запросов от nginx

5.2 Изменено значение переменной KEYCLOAK_PORT
| было | стало |
|------|-------|
| 1443 | 8081 |

Причина: Keycloack не слушает порт 1443, в образе зашит порт 8081

6. В docker compose добавлены стандартные переменные для сервиса quotation-book-pgadmin
      PGADMIN_DEFAULT_EMAIL: "${PG_ADMIN_USER}"
      PGADMIN_DEFAULT_PASSWORD: "${PG_ADMIN_PASSWORD}"

Причина: Переменные отсутсовали в файле .env

7. В docker compose добавлены стандартные переменные для сервиса quotation-book-rabbitmq
В .env отсутствовали переменные, которые используются в образе и влияют на подключение quotation-book-backend-worker
      ENABLE_SSL: "${ENABLE_SSL_RABBITMQ}"
      SSL_PORT: "${RABBITMQ_PORT_SSL}"
      CA_CERT_FILE: "${CA_CERTIFICATE}"
      CERT_FILE: "${RABBITMQ_CERT_FILE}"
      KEY_FILE: "${RABBITMQ_KEY_FILE}"

Причина: Переменные отсутсовали в файле .env. Переменные используются в docker-образе и влияют на подключение сервиса quotation-book-backend-worker

8. Изменён файл конфигурации nginx "default-location-conf.template" для сервиса quotation-book-frontend
| было | стало |
|----------------------------------------------------------|------------------------|
| location /keycloak/ {
    proxy_pass https://${KEYCLOAK_HOST}:${KEYCLOAK_PORT}/; | location /keycloak/ {
    proxy_pass http://${KEYCLOAK_HOST}:${KEYCLOAK_PORT}/; |

| location /static/ {
    alias /opt/app-root/static/ | location /static/ {
    alias /opt/app-root/src/static/ |

Причина:
1. Keycloak настроен на работу по протоколу http на 8081 порту
2. Ошибка в alias, адрес директории отличается от dockerfile QB.Frontend.Dockerfile

9. Изменены маршруты аутентификации в файле auth.py
| было | стало |
|-------|------|
| token_url = f"https | token_url = f"http |
| userinfo_url = f"https | userinfo_url = f"http |
| token_refresh_url = f"https | token_refresh_url = f"http |

Причина: Необходимо для корректного формирования url для keycloak по http

10. В docker compose переопределена переменная REDIS_PORT для сервиса quotation-book-backend-worker
      TLS_PORT: "${REDIS_PORT_TLS}"
      ENABLE_SSL: "${ENABLE_SSL_REDIS}"
      TLS_CERT_FILE: "${TLS_CERT_FILE_REDIS}"
      TLS_KEY_FILE: "${TLS_KEY_FILE_REDIS}"
      TLS_CLIENT_CERT_AUTH: "${TLS_CLIENT_CERT_AUTH}"
      TLS_CA_CERT_FILE: "${CA_CERTIFICATE}"

Причина: Несоответствие переменных в docker-entrypoint.sh и файле .env

11. В docker compose переопределены переменные для сервисов quotation-book-backend-*
      DB_HOST: "${POSTGRES_DB_HOST}"
      DB_PORT: "${POSTGRES_DB_PORT}" 

Причина: Несоответствие переменных в docker-entrypoint.sh и файле .env

12. В docker compose переопределены переменные для сервиса quotation-book-java-obs
      DB_HOST: "${POSTGRES_DB_HOST}"
      DB_PORT: "${POSTGRES_DB_PORT}"
      DB_NAME: "${POSTGRES_DB_NAME}"
      DB_USER: "${POSTGRES_DB_USER}"
      DB_PASSWORD: "${POSTGRES_DB_PASSWORD}"

Причина: Несоответствие переменных в MetricsApp.java и файле .env
