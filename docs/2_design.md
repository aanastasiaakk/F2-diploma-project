# Розділ 2. Проєктування архітектури системи

## 2.1 Контекстне та контейнерне моделювання системи (C4 Model)

### C4 Level 1 — System Context Diagram

```mermaid
flowchart LR
    classDef person fill:#1a3a6b,color:#fff,stroke:#0b1f3d;
    classDef system fill:#2a63b7,color:#fff,stroke:#154a8f;
    classDef external fill:#8a8a8a,color:#fff,stroke:#5c5c5c;

    user["Користувач<br/>[Person]<br/>Власник гардероба"]:::person
    stylesync["StyleSync<br/>[Software System]<br/>Вебсистема підбору гардероба"]:::system
    email["Email Provider<br/>[Software System]<br/>Зовнішній SMTP-сервіс"]:::external

    user -->|"Завантажує фото, отримує рекомендації<br/>[HTTPS/REST]"| stylesync
    stylesync -->|"Надсилає листи<br/>[SMTP/TLS]"| email
```

Опис вузлів: актор **Користувач** — єдиний тип кінцевого користувача системи, взаємодіє через веб-браузер. Система **StyleSync** — межі проєкту кваліфікаційної роботи. Зовнішня система **Email Provider** — єдина стороння залежність MVP, винесена за межі системи.

### C4 Level 2 — Container Diagram

```mermaid
flowchart TB
    classDef person fill:#1a3a6b,color:#fff,stroke:#0b1f3d;
    classDef container fill:#2a63b7,color:#fff,stroke:#154a8f;
    classDef external fill:#8a8a8a,color:#fff,stroke:#5c5c5c;

    user["Користувач<br/>[Person]"]:::person

    subgraph stylesync["StyleSync"]
        spa["Web SPA<br/>[React, TypeScript]<br/>Інтерфейс користувача"]:::container
        gateway["Reverse Proxy / Gateway<br/>[Nginx]<br/>TLS termination, Rate Limiting"]:::container
        api["Backend API<br/>[FastAPI]<br/>Автентифікація, гардероб, рекомендації"]:::container
        mlservice["ML Inference Service<br/>[PyTorch, CLIP]<br/>Класифікація, embeddings"]:::container
        worker["Celery Worker Pool<br/>[Celery]<br/>Споживач черги CV-завдань"]:::container
        postgres[("PostgreSQL<br/>[+pgvector]<br/>Користувачі, гардероб, embeddings")]:::container
        redis[("Redis<br/>Кеш рекомендацій")]:::container
        rabbitmq[["RabbitMQ<br/>Черга CV-завдань"]]:::container
        objectstorage[("Object Storage<br/>[MinIO]<br/>Фотографії речей")]:::container
        observability["Observability Stack<br/>[Prometheus, Grafana]<br/>Метрики і моніторинг"]:::container
    end

    email["Email Provider<br/>[SMTP]"]:::external

    user -->|HTTPS| spa
    spa -->|"HTTPS/REST"| gateway
    gateway -->|"HTTP/REST"| api
    api -->|"SQL/TCP"| postgres
    api -->|"Redis/TCP"| redis
    api -->|"S3 API/HTTPS"| objectstorage
    api -->|"AMQP 0-9-1"| rabbitmq
    worker -->|"AMQP 0-9-1"| rabbitmq
    worker -->|"gRPC/HTTP2"| mlservice
    worker -->|"SQL/TCP"| postgres
    api -->|"SMTP/TLS"| email
    api -->|HTTP| observability
    worker -->|HTTP| observability
```

Протоколи взаємодії: SPA → Gateway (HTTPS/REST), Gateway → Backend API (HTTP/REST внутрішньо), Backend API → PostgreSQL (SQL/TCP), Backend API → Redis (Redis Protocol/TCP), Backend API → Object Storage (S3 API/HTTPS), Backend API → RabbitMQ (AMQP 0-9-1), Worker → ML Inference Service (gRPC/HTTP2), Backend API → Email Provider (SMTP/TLS).

Трасування вимог: контейнер **Backend API** [C4 Container](#c4-level-2-container-diagram) реалізує функціональні вимоги [SRS-FUN-01](./srs.md#2-functional-requirements), [SRS-FUN-02a](./srs.md#2-functional-requirements), [SRS-FUN-05](./srs.md#2-functional-requirements), [SRS-FUN-06](./srs.md#2-functional-requirements) та базується на рішенні [ADR-001](./adr/0001-choice-of-technology.md) щодо зберігання embeddings у PostgreSQL. Контейнер **ML Inference Service** реалізує [SRS-FUN-03](./srs.md#2-functional-requirements). Контейнер **Reverse Proxy / API Gateway** реалізує нефункціональну вимогу [SRS-NFR-SEC-03](./srs.md#3-non-functional-requirements) (Rate Limiting). Контейнер **RabbitMQ + Celery Worker Pool** реалізує [SRS-NFR-REL-01](./srs.md#3-non-functional-requirements) та [SRS-NFR-REL-02](./srs.md#3-non-functional-requirements) (retry та Circuit Breaker).

---

## 2.2 Компонентне моделювання критичного сервісу системи (C4 Model: Component)

Обрано для деталізації **ML Inference Service** — найскладніший за обчислювальним навантаженням контейнер системи.

```mermaid
flowchart TB
    classDef comp fill:#2a63b7,color:#fff,stroke:#154a8f;
    classDef external fill:#8a8a8a,color:#fff,stroke:#5c5c5c;

    subgraph mlservice["ML Inference Service"]
        grpc_server["gRPC Server<br/>[grpc.aio]<br/>Приймає запити від Worker Pool"]:::comp
        validation["Validation Middleware<br/>[Pydantic/Protobuf]<br/>Перевіряє формат і розмір"]:::comp
        error_handler["Global Error Handler<br/>[gRPC interceptor]<br/>Централізована обробка помилок"]:::comp
        classifier["ImageClassifierService<br/>[PyTorch, CLIP]<br/>Тип, колір, сезонність"]:::comp
        embedder["EmbeddingGeneratorService<br/>[PyTorch]<br/>Формує embedding"]:::comp
        preprocessor["ImagePreprocessor<br/>[torchvision]<br/>Нормалізація, ресайз"]:::comp
        storage_iface["IObjectStorageClient<br/>[Protocol]<br/>Абстракція сховища (DI)"]:::comp
        minio_client["MinIOStorageClient<br/>[boto3/minio-py]<br/>Реалізація клієнта S3"]:::comp
        metrics["Metrics Interceptor<br/>[prometheus-client]<br/>Збір метрик телеметрії"]:::comp
    end

    objectstorage["Object Storage<br/>[MinIO]"]:::external

    grpc_server -->|"in-process"| validation
    validation -->|"in-process"| error_handler
    grpc_server -->|"in-process"| classifier
    classifier -->|"in-process"| preprocessor
    preprocessor -->|"interface call"| storage_iface
    storage_iface -->|DI| minio_client
    minio_client -->|"S3 API/HTTPS"| objectstorage
    classifier -->|"in-process"| embedder
    grpc_server -->|"in-process"| metrics
```

Опис компонентів: **gRPC Server** — вхідний шар (Inbound), приймає запити виключно від Celery Worker Pool у межах довіреної внутрішньої мережі (публічна автентифікація JWT виконується на рівні Reverse Proxy/Backend API, повторна перевірка JWT на цьому внутрішньому сервісі визнана надлишковою для обсягу MVP — внутрішня мережа ізольована мережевим рівнем Docker). **Request Validation Middleware** та **Global Error Handler** — шар захисту вхідних даних, еквівалент Guards/Middlewares. **ImageClassifierService** та **EmbeddingGeneratorService** — шар бізнес-логіки (Core/Services). **IObjectStorageClient** — інтерфейсна абстракція, що реалізує принцип інверсії залежностей (Dependency Inversion Principle): **ImagePreprocessor** залежить від абстракції, а не від конкретної реалізації **MinIOStorageClient**, що дозволяє підміняти сховище без зміни бізнес-логіки. **Metrics Interceptor** — наскрізний компонент телеметрії.

Трасування: цей розподіл компонентів реалізує нефункціональну вимогу [SRS-NFR-PERF-02](./srs.md#3-non-functional-requirements) (латентність обробки CV-завдання) та спирається на рішення [ADR-001](./adr/0001-choice-of-technology.md) щодо відокремлення зберігання даних від обчислювальної логіки.

## 2.3 Документування архітектурних рішень (Architecture Decision Records)

Повний текст рішення — [ADR-001: Вибір сховища для векторних embeddings](./adr/0001-choice-of-technology.md) (статус: **Approved**).

## 2.4. Проєктування та моделювання баз даних

```mermaid
erDiagram
    USERS ||--o{ WARDROBE_ITEMS : owns
    USERS ||--o{ OUTFITS : creates
    USERS ||--o{ RECOMMENDATIONS : receives
    USERS ||--o{ SESSION_LOGS : generates
    OUTFITS ||--o{ OUTFIT_ITEMS : contains
    WARDROBE_ITEMS ||--o{ OUTFIT_ITEMS : "included in"
    OUTFITS ||--o{ RECOMMENDATIONS : "suggested as"
    RECOMMENDATIONS ||--o{ RATINGS : "rated by"

    USERS {
        uuid id PK
        varchar email "UNIQUE, NOT NULL"
        varchar password_hash
        int version "optimistic locking"
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at "nullable, soft delete"
    }
    WARDROBE_ITEMS {
        uuid id PK
        uuid user_id FK
        varchar category "INDEX"
        varchar color
        varchar season
        text image_url
        vector_512 embedding "HNSW INDEX"
        enum status "PROCESSING|READY|FAILED"
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at "nullable, soft delete"
    }
    OUTFITS {
        uuid id PK
        uuid user_id FK "INDEX"
        enum status "DRAFT|ACTIVE|ARCHIVED"
        timestamptz created_at
        timestamptz updated_at
    }
    OUTFIT_ITEMS {
        uuid outfit_id PK_FK
        uuid item_id PK_FK
    }
    RECOMMENDATIONS {
        uuid id PK
        uuid user_id FK "INDEX"
        uuid outfit_id FK
        float score
        enum status "PENDING|RATED"
        timestamptz created_at
    }
    RATINGS {
        uuid id PK
        uuid recommendation_id FK "INDEX"
        boolean liked
        timestamptz created_at
    }
    SESSION_LOGS {
        uuid id PK
        uuid user_id FK "INDEX"
        varchar action
        jsonb metadata
        timestamptz created_at
    }
```

**Специфікація полів (вибірково):**

| Таблиця | Поле | Тип | Обмеження | Опис |
|---|---|---|---|---|
| users | id | UUID | PK | Унікальний ідентифікатор користувача |
| users | email | VARCHAR(255) | UNIQUE, NOT NULL | Логін користувача |
| users | version | INT | DEFAULT 1 | Оптимістичне блокування при конкурентних оновленнях профілю |
| wardrobe_items | embedding | VECTOR(512) | HNSW INDEX | Векторне представлення зображення для пошуку схожості (pgvector) |
| wardrobe_items | status | ENUM | PROCESSING/READY/FAILED | Стан асинхронної ML-обробки речі |
| wardrobe_items | deleted_at | TIMESTAMPTZ | NULLABLE | Soft delete — збереження історії образів, де річ вже використана |
| outfits | status | ENUM | DRAFT/ACTIVE/ARCHIVED | Життєвий цикл сформованого образу |
| outfit_items | (outfit_id, item_id) | UUID, UUID | COMPOSITE PK | Унеможливлює дублювання речі в тому самому образі |
| recommendations | status | ENUM | PENDING/RATED | Чи вже оцінена рекомендація користувачем |

Трасування: схема реалізує функціональні вимоги [SRS-FUN-02a, SRS-FUN-04, SRS-FUN-05, SRS-FUN-06, SRS-FUN-07](./srs.md) та деталізує контейнер **PostgreSQL [+pgvector]** з підрозділу 2.1.

## 2.5. Специфікація міжсервісних контрактів (API)

Деталізовану специфікацію ендпоінтів взаємодії Web SPA → Backend API у форматі OpenAPI 3.0 зафіксовано у файлі [openapi.yaml](./api/openapi.yaml). Контракт покриває автентифікацію (OAuth 2.0/OIDC, JWT Bearer), завантаження речі гардероба на асинхронну обробку та отримання персоналізованих рекомендацій, з повною схемою обробки помилок (400/401/429) та механізмом авторизації.

## 2.6. Моделювання логіки критичних вузлів системи

### Алгоритм 1. Пайплайн класифікації фото та генерації embedding (ML Inference)

```mermaid
graph TD
    A([Початок: Worker отримав Task із черги]) --> B["image_ref := Task.image_url"]
    B --> C{"image_ref = NULL OR image_ref = ''"}
    C -->|TRUE| C1["status := FAILED<br/>error_log := EMPTY_IMAGE_REF"]
    C1 --> Z([Кінець])
    C -->|FALSE| D["raw_image := ObjectStorage.Fetch(image_ref)"]
    D --> E{"raw_image = NULL"}
    E -->|TRUE| F["retry_count := retry_count + 1"]
    F --> G{"NOT (retry_count <= MAX_RETRIES)"}
    G -->|TRUE| G1["status := FAILED<br/>error_log := STORAGE_UNAVAILABLE"]
    G1 --> Z
    G -->|FALSE| D
    E -->|FALSE| H["result := MLService.Classify(raw_image)"]
    H --> I{"MLService.available = FALSE OR result.error <> NULL"}
    I -->|TRUE| I1["circuit_breaker_state := OPEN<br/>status := FAILED"]
    I1 --> Z
    I -->|FALSE| J["embedding[i] := result.embedding<br/>category[i] := result.category<br/>color[i] := result.color"]
    J --> K["DB.Save(WardrobeItem.id, embedding[i], category[i], color[i], status := READY)"]
    K --> Z
```

### Алгоритм 2. Розрахунок персоналізованих рекомендацій

```mermaid
graph TD
    A([Початок: запит GET /recommendations]) --> B{"COUNT(WardrobeItems[user]) = 0"}
    B -->|TRUE| B1["recommendations := DEFAULT_POPULAR_LIST<br/>status := FALLBACK_COLD_START"]
    B1 --> Z([Кінець: повернути recommendations])
    B -->|FALSE| C["user_vector := AVG(embedding[i] FOR i WHERE Outfit_Rating[i].liked = TRUE)"]
    C --> D{"user_vector = NULL"}
    D -->|TRUE| E["user_vector := AVG(embedding[i] FOR ALL i)"]
    D -->|FALSE| F["candidates := Outfits[user]"]
    E --> F
    F --> G{"COUNT(candidates) = 0"}
    G -->|TRUE| G1["recommendations := EMPTY<br/>status := NO_CANDIDATES"]
    G1 --> Z
    G -->|FALSE| H["i := 0"]
    H --> I{"i < COUNT(candidates)"}
    I -->|TRUE| J["similarity[i] := CosineSimilarity(user_vector, candidates[i].embedding)"]
    J --> K{"EXISTS(Ratings[candidates[i]]) AND Ratings[candidates[i]].liked = TRUE"}
    K -->|TRUE| K1["bonus[i] := 0.1"]
    K -->|FALSE| L{"EXISTS(Ratings[candidates[i]]) AND Ratings[candidates[i]].liked = FALSE"}
    L -->|TRUE| L1["bonus[i] := -0.1"]
    L -->|FALSE| L2["bonus[i] := 0"]
    K1 --> M["score[i] := similarity[i] + bonus[i]"]
    L1 --> M
    L2 --> M
    M --> N["i := i + 1"]
    N --> I
    I -->|FALSE| O["recommendations := SORT(candidates BY score DESC)"]
    O --> Z
```

Трасування: Алгоритм 1 реалізує [SRS-FUN-03, SRS-FUN-04](./srs.md) та [SRS-NFR-REL-01/02](./srs.md) (retry-політика, Circuit Breaker). Алгоритм 2 реалізує [SRS-FUN-05, SRS-FUN-06](./srs.md).

## 2.7. Інтеграційний аудит та верифікація цілісності доменів

**Виявлені розбіжності:**

1. **API-to-Database Mismatch.** Схема `WardrobeItem` в OpenAPI-контракті (2.5) містить поле `status: PROCESSING|READY|FAILED`, а в початковій ERD-схемі (2.4, до рефакторингу) поле `status` у таблиці `WARDROBE_ITEMS` було відсутнє.
2. **Logic-to-Data Mismatch.** Алгоритм 2 (2.6) у початковій версії звертався до поля `liked` безпосередньо на `WARDROBE_ITEMS`, тоді як за ERD-схемою оцінка (`RATINGS.liked`) прив'язана до `RECOMMENDATIONS`/`OUTFITS`.

**Усунення розбіжностей:** у фінальну ERD-схему (2.4) додано поле `status ENUM(PROCESSING, READY, FAILED)` до `WARDROBE_ITEMS`; формулу Алгоритму 2 (2.6) скориговано — `user_vector` обчислюється через `Outfit_Rating[i].liked` (зв'язок `OUTFITS → RECOMMENDATIONS → RATINGS`).

**Production Readiness Report:** критичних блокерів (Showstoppers) не виявлено. Специфікації даних, API-контрактів та алгоритмів синхронізовані між собою за типами, іменами сутностей та життєвими циклами статусів. Архітектурні артефакти Розділу 2 готові до передачі в розробку (Розділ 3).
