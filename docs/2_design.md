# Розділ 2. Проєктування архітектури системи

## 2.1 Контекстне та контейнерне моделювання системи (C4 Model)

### C4 Level 1 — System Context Diagram

```mermaid
C4Context
    title System Context Diagram — StyleSync

    Person(user, "Користувач", "Власник гардероба: завантажує фото речей, переглядає рекомендації, надає фідбек")

    System(stylesync, "StyleSync", "Вебсистема персоналізованого підбору та управління гардеробом")

    System_Ext(email_ext, "Email Provider", "Зовнішній SMTP-сервіс для транзакційних листів")

    Rel(user, stylesync, "Завантажує фото, переглядає рекомендації, надає фідбек", "HTTPS/REST")
    Rel(stylesync, email_ext, "Надсилає листи підтвердження реєстрації", "SMTP/TLS")
```

Опис вузлів: актор **Користувач** — єдиний тип кінцевого користувача системи, взаємодіє через веб-браузер. Система **StyleSync** — межі проєкту кваліфікаційної роботи. Зовнішня система **Email Provider** — єдина стороння залежність MVP, винесена за межі системи.

### C4 Level 2 — Container Diagram

```mermaid
C4Container
    title Container Diagram — StyleSync

    Person(user, "Користувач")

    System_Boundary(stylesync, "StyleSync") {
        Container(spa, "Web SPA", "React, TypeScript, TailwindCSS", "Інтерфейс користувача")
        Container(gateway, "Reverse Proxy / API Gateway", "Nginx", "TLS termination, Rate Limiting, маршрутизація запитів")
        Container(api, "Backend API", "Python, FastAPI", "Бізнес-логіка: автентифікація, гардероб, рекомендації")
        Container(mlservice, "ML Inference Service", "Python, PyTorch, CLIP", "Класифікація зображень та генерація embeddings")
        Container(worker, "Celery Worker Pool", "Python, Celery", "Споживачі черги асинхронних CV-завдань")
        ContainerDb(postgres, "PostgreSQL", "PostgreSQL 16 + pgvector", "Користувачі, гардероб, embeddings, рекомендації")
        ContainerDb(redis, "Redis", "Redis 7", "Кеш рекомендацій")
        ContainerQueue(rabbitmq, "RabbitMQ", "AMQP 0-9-1", "Черга асинхронних CV-завдань")
        ContainerDb(objectstorage, "Object Storage", "MinIO (S3-compatible)", "Зберігання фотографій речей")
        Container(observability, "Observability Stack", "Prometheus, Grafana", "Збір метрик і моніторинг доступності")
    }

    System_Ext(email_ext, "Email Provider", "SMTP")

    Rel(user, spa, "Використовує", "HTTPS")
    Rel(spa, gateway, "Запити до API", "HTTPS/REST")
    Rel(gateway, api, "Проксіює запити", "HTTP/REST")
    Rel(api, postgres, "Читає/записує дані", "SQL/TCP")
    Rel(api, redis, "Читає кешовані рекомендації", "Redis Protocol/TCP")
    Rel(api, objectstorage, "Завантажує/отримує фото", "S3 API/HTTPS")
    Rel(api, rabbitmq, "Публікує завдання CV-аналізу", "AMQP 0-9-1")
    Rel(worker, rabbitmq, "Споживає завдання", "AMQP 0-9-1")
    Rel(worker, mlservice, "Викликає інференс моделі", "gRPC/HTTP2")
    Rel(worker, postgres, "Записує embeddings і статус", "SQL/TCP")
    Rel(api, email_ext, "Надсилає листи", "SMTP/TLS")
    Rel(api, observability, "Експортує метрики", "HTTP (Prometheus scrape)")
    Rel(worker, observability, "Експортує метрики", "HTTP (Prometheus scrape)")
```

Протоколи взаємодії: SPA → Gateway (HTTPS/REST), Gateway → Backend API (HTTP/REST внутрішньо), Backend API → PostgreSQL (SQL/TCP), Backend API → Redis (Redis Protocol/TCP), Backend API → Object Storage (S3 API/HTTPS), Backend API → RabbitMQ (AMQP 0-9-1), Worker → ML Inference Service (gRPC/HTTP2), Backend API → Email Provider (SMTP/TLS).

Трасування вимог: контейнер **Backend API** [C4 Container](#c4-level-2-container-diagram) реалізує функціональні вимоги [SRS-FUN-01](./srs.md#2-functional-requirements), [SRS-FUN-02a](./srs.md#2-functional-requirements), [SRS-FUN-05](./srs.md#2-functional-requirements), [SRS-FUN-06](./srs.md#2-functional-requirements) та базується на рішенні [ADR-001](./adr/0001-choice-of-technology.md) щодо зберігання embeddings у PostgreSQL. Контейнер **ML Inference Service** реалізує [SRS-FUN-03](./srs.md#2-functional-requirements). Контейнер **Reverse Proxy / API Gateway** реалізує нефункціональну вимогу [SRS-NFR-SEC-03](./srs.md#3-non-functional-requirements) (Rate Limiting). Контейнер **RabbitMQ + Celery Worker Pool** реалізує [SRS-NFR-REL-01](./srs.md#3-non-functional-requirements) та [SRS-NFR-REL-02](./srs.md#3-non-functional-requirements) (retry та Circuit Breaker).

---

## 2.2 Компонентне моделювання критичного сервісу системи (C4 Model: Component)

Обрано для деталізації **ML Inference Service** — найскладніший за обчислювальним навантаженням контейнер системи.

```mermaid
C4Component
    title Component Diagram — ML Inference Service

    Container_Boundary(mlservice, "ML Inference Service") {
        Component(grpc_server, "gRPC Server", "grpc.aio", "Приймає запити класифікації від Celery Worker Pool")
        Component(validation, "Request Validation Middleware", "Pydantic/Protobuf validators", "Перевіряє формат, розмір і MIME-тип зображення")
        Component(error_handler, "Global Error Handler", "gRPC interceptor", "Централізована обробка винятків, формування кодів помилок gRPC")
        Component(classifier, "ImageClassifierService", "PyTorch, CLIP", "Інференс моделі: тип, колір, сезонність")
        Component(embedder, "EmbeddingGeneratorService", "PyTorch", "Формування векторного представлення (embedding)")
        Component(preprocessor, "ImagePreprocessor", "torchvision.transforms", "Нормалізація та ресайз вхідного зображення")
        Component(storage_iface, "IObjectStorageClient", "Python Protocol/ABC", "Абстракція доступу до сховища фото (Dependency Inversion)")
        Component(minio_client, "MinIOStorageClient", "boto3/minio-py", "Конкретна реалізація клієнта S3-сумісного сховища")
        Component(metrics, "Metrics Interceptor", "prometheus-client", "Збір метрик латентності та кількості інференсів")
    }

    ContainerDb_Ext(objectstorage, "Object Storage", "MinIO (S3-compatible)")

    Rel(grpc_server, validation, "Передає запит на валідацію", "in-process call")
    Rel(validation, error_handler, "Делегує обробку невалідних запитів", "in-process call")
    Rel(grpc_server, classifier, "Передає валідний запит", "in-process call")
    Rel(classifier, preprocessor, "Запитує попередньо оброблене зображення", "in-process call")
    Rel(preprocessor, storage_iface, "Отримує байти зображення", "interface call (DI)")
    Rel(storage_iface, minio_client, "Реалізовано через", "Dependency Injection")
    Rel(minio_client, objectstorage, "Завантажує файл", "S3 API/HTTPS")
    Rel(classifier, embedder, "Передає тензор ознак", "in-process call")
    Rel(grpc_server, metrics, "Логує тривалість обробки", "in-process call")
```

Опис компонентів: **gRPC Server** — вхідний шар (Inbound), приймає запити виключно від Celery Worker Pool у межах довіреної внутрішньої мережі (публічна автентифікація JWT виконується на рівні Reverse Proxy/Backend API, повторна перевірка JWT на цьому внутрішньому сервісі визнана надлишковою для обсягу MVP — внутрішня мережа ізольована мережевим рівнем Docker). **Request Validation Middleware** та **Global Error Handler** — шар захисту вхідних даних, еквівалент Guards/Middlewares. **ImageClassifierService** та **EmbeddingGeneratorService** — шар бізнес-логіки (Core/Services). **IObjectStorageClient** — інтерфейсна абстракція, що реалізує принцип інверсії залежностей (Dependency Inversion Principle): **ImagePreprocessor** залежить від абстракції, а не від конкретної реалізації **MinIOStorageClient**, що дозволяє підміняти сховище без зміни бізнес-логіки. **Metrics Interceptor** — наскрізний компонент телеметрії.

Трасування: цей розподіл компонентів реалізує нефункціональну вимогу [SRS-NFR-PERF-02](./srs.md#3-non-functional-requirements) (латентність обробки CV-завдання) та спирається на рішення [ADR-001](./adr/0001-choice-of-technology.md) щодо відокремлення зберігання даних від обчислювальної логіки.

## 2.3 Документування архітектурних рішень (Architecture Decision Records)

Повний текст рішення — [ADR-001: Вибір сховища для векторних embeddings](./adr/0001-choice-of-technology.md) (статус: **Approved**).
